---
title: MACE多卡训练无法保存模型
published: 2026-03-19
description: "有关于Mace 使用pytorch进行多卡训练时出现无法保存模型异常的排查与修复"
tags: ["GPU", "Mace", "pytoch"]
category: Debug
draft: false
---

## 环境

- Python 3.11
- PyTorch（通过 `torchrun` 进行 DDP 分布式多进程训练）
- e3nn（含 `@compile_mode("script")` 装饰器）
- CP-MACE（`deps/CP-MACE/`）

## 问题描述

使用 CP-MACE 进行多卡（DDP）训练时，训练过程本身正常完成，但在训练结束后的模型保存阶段崩溃，抛出：

```
_pickle.PickleError: ScriptFunction cannot be pickled
```

## 排查过程

### 1. 现象：训练完成后崩溃

WandB 上观察到模型总是在训练成功结束后立即崩溃。将 `max_num_epochs` 设为 1 以快速复现，日志显示训练本身能正常启动和收敛：

```bash
2026-03-18 22:28:11.051 INFO: Using gradient clipping with tolerance=10.000
2026-03-18 22:28:11.052 INFO: ===========TRAINING===========
2026-03-18 22:28:11.052 INFO: Started training, reporting errors on validation set
2026-03-18 22:28:11.052 INFO: Loss metrics on validation set
2026-03-18 22:28:42.950 INFO: Initial: head: default, loss=0.83482060, RMSE_E_per_atom= 949.164 meV, RMSE_F= 267.246 meV / A, RMSE_P=  0.0069 V,

# 后续将没有内容了，wandb上status显示crashed
```

初步判断问题出在训练完成后的收尾阶段。

### 2. 定位到 DDP 分布式进程异常

查看 torchrun 日志，发现子进程抛出了 `ChildFailedError`：

```txt
Traceback (most recent call last):
  File "/.conda/envs/catdt/bin/torchrun", line 6, in <module>
    sys.exit(main())
  File ".../torch/distributed/run.py"
    elastic_launch(...)
  File ".../torch/distributed/launcher/api.py"
    return launch_agent(self._config, self._entrypoint, list(args))
  ...
torch.distributed.elastic.multiprocessing.errors.ChildFailedError:
```

该错误是 torchrun 对子进程内部异常的包装——注意 DDP 使用的是**多进程**，具体错误原因被隐藏在内层堆栈中，排查难度较大。

### 3. 定位到 `deepcopy(model)`

进一步追踪 rank0 的详细堆栈，在测试将 `deepcopy(model)` 单独拎出后，最终确认真正的异常出在 `run_train.py` 模型保存阶段调用的该深拷贝函数：

```txt
_pickle.PickleError: ScriptFunction cannot be pickled

[rank0]: Traceback (most recent call last):
[rank0]:   File "deps/CP-MACE/mace/cli/run_train.py", line 865, in run
[rank0]:     model_to_save = deepcopy(model)
[rank0]:                     ^^^^^^^^^^^^^^^
[rank0]:   File "copy.py", line 172, in deepcopy
[rank0]:     y = _reconstruct(x, memo, *rv)
[rank0]:   File "copy.py", line 271, in _reconstruct
[rank0]:     state = deepcopy(state, memo)
[rank0]:   File "copy.py", line 231, in _deepcopy_dict
[rank0]:     y[deepcopy(key, memo)] = deepcopy(value, memo)
              ...（递归遍历模型 __dict__）...
[rank0]:   File "torch/jit/_script.py", line 71, in _reduce
[rank0]:     raise pickle.PickleError("ScriptFunction cannot be pickled")
```

## 根因分析

### `@compile_mode("script")` 与 e3nn codegen

CP-MACE 的模型类及其子模块使用了 e3nn 的 `@compile_mode("script")` 装饰器：

```python
# deps/CP-MACE/mace/modules/models.py

@compile_mode("script")       # line 111
class MACE(torch.nn.Module):
    ...

@compile_mode("script")       # line 418
class ScaleShiftMACE(MACE):
    ...
```

当 e3nn 的 `jit_script_fx` 优化选项处于启用状态（默认行为）时，模型在实例化过程中会通过 e3nn codegen 将部分方法编译为 `torch.jit.ScriptFunction` 对象。

### 为何 `deepcopy` 失败

Python 的 `copy.deepcopy()` 内部依赖 pickle 协议进行序列化。而 `torch.jit.ScriptFunction` 显式禁止了 pickle：

```python
# torch/jit/_script.py
def _reduce(self):
    raise pickle.PickleError("ScriptFunction cannot be pickled")
```

因此，当模型内部包含 `ScriptFunction` 对象时，`deepcopy(model)` 必然失败。

### 故障链

```
模型实例化（e3nn codegen 默认启用）
  → @compile_mode("script") 使模型内部生成 ScriptFunction 对象
    → 训练正常完成，进入保存阶段
      → deepcopy(model) 被调用，试图创建模型副本
        → deepcopy 递归遍历模型 __dict__
          → 遇到 ScriptFunction，触发 pickle 序列化
            → ScriptFunction._reduce() 抛出 PickleError
              → 异常未被捕获，rank0 进程崩溃
                → torchrun 抛出 ChildFailedError
```

## 修复方案

### 核心思路

不使用 `deepcopy`，而是**在 e3nn codegen 关闭的上下文中重新构建一个不含 `ScriptFunction` 的干净模型**，再通过 `load_state_dict()` 加载训练好的权重。

e3nn 提供了 `disable_e3nn_codegen()` 上下文管理器（位于 `mace/tools/compile.py`），在该上下文中创建的模型不会生成 `ScriptFunction` 对象：

```python
# mace/tools/compile.py
@contextmanager
def disable_e3nn_codegen():
    init_val = get_optimization_defaults()["jit_script_fx"]
    set_optimization_defaults(jit_script_fx=False)
    yield
    set_optimization_defaults(jit_script_fx=init_val)
```

### 修改文件

`deps/CP-MACE/mace/cli/run_train.py`，模型保存阶段（`for swa_eval in swas:` 循环内，`if rank == 0:` 分支）。

### 修改前（原始代码）

```python
        if rank == 0:
            # Save entire model
            if swa_eval:
                model_path = Path(args.checkpoints_dir) / (tag + "_stagetwo.model")
            else:
                model_path = Path(args.checkpoints_dir) / (tag + ".model")
            logging.info(f"Saving model to {model_path}")
            model_to_save = deepcopy(model)          # ← 此处崩溃
            if args.enable_cueq:
                model_to_save = run_cueq_to_e3nn(deepcopy(model), device=device)
```

### 修改后

```python
        if rank == 0:
            # 在 e3nn codegen 关闭的上下文中重建模型，避免生成 ScriptFunction
            with disable_e3nn_codegen():
                model_to_save, _ = configure_model(
                    args, train_loader, atomic_energies,
                    model_foundation, heads, z_table,
                )
            model_to_save.to(device)
            model_to_save.load_state_dict(model.state_dict())

            # Save entire model
            if swa_eval:
                model_path = Path(args.checkpoints_dir) / (tag + "_stagetwo.model")
            else:
                model_path = Path(args.checkpoints_dir) / (tag + ".model")
            logging.info(f"Saving model to {model_path}")
            if args.enable_cueq:
                model_to_save = run_cueq_to_e3nn(model_to_save, device=device)
```

### 修改要点

| # | 修改内容 | 说明 |
|---|---------|------|
| 1 | 移除 `model_to_save = deepcopy(model)` | 不再对含有 `ScriptFunction` 的模型做深拷贝 |
| 2 | 新增 `disable_e3nn_codegen()` + `configure_model()` | 重建一个结构相同但不含 `ScriptFunction` 的干净模型 |
| 3 | 新增 `load_state_dict(model.state_dict())` | 将训练好的参数从原模型复制到新模型 |
| 4 | CUEQ 转换改用 `model_to_save` | 新模型已是独立副本，无需再 `deepcopy` |

## 附录：涉及 `@compile_mode("script")` 的类

以下所有类在 e3nn codegen 启用时均会在实例化过程中产生 `ScriptFunction`，均可能受此问题影响：

**模型层**（`mace/modules/models.py`）：
- `MACE`、`ScaleShiftMACE`、`AtomicDipolesMACE`、`EnergyDipolesMACE`

**构建模块**（`mace/modules/blocks.py`）：
- `LinearNodeEmbeddingBlock`、`LinearReadoutBlock`、`NonLinearReadoutBlock`、`LinearDipoleReadoutBlock`、`NonLinearDipoleReadoutBlock`、`AtomicEnergiesBlock`、`RadialEmbeddingBlock`、`EquivariantProductBasisBlock`、`InteractionBlock`、`TensorProductWeightsBlock` 等共 19 个 Block 类

**径向基函数**（`mace/modules/radial.py`）：
- `BesselBasis`、`ChebychevBasis`、`GaussianBasis`、`PolynomialCutoff`、`ZBLBasis`、`AgnesiTransform`、`SoftTransform`
