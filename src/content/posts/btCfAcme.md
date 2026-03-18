---
title: 宝塔使用Cloudflare API Token部署ACME
published: 2026-03-09
description: "宝塔使用pyhton，搭配Cloudflare API Token部署ACME"
tags: ["教程", "宝塔", "ACME"]
category: Guides
draft: false
---

## 目录
- [1. 项目简介](#1-项目简介)
- [2. 实现原理](#2-实现原理)
- [3. 部署前准备](#3-部署前准备)
- [4. 在宝塔中部署 Python 服务](#4-在宝塔中部署-python-服务)
- [5. 初始化 SSL 状态（建议）](#5-初始化-ssl-状态建议)
- [6. 配置计划任务触发更新](#6-配置计划任务触发更新)
- [7. 验证部署是否成功](#7-验证部署是否成功)
- [8. 环境变量说明（完整）](#8-环境变量说明完整)
- [9. 常见问题与排查](#9-常见问题与排查)

## 1. 项目简介
宝塔面板当前内置的 Let's Encrypt DNS 验证方式，目前11版本依旧只能使用 Cloudflare Global API Key作为DNS解析管理的令牌。私以为该方案权限过大，不利于最小权限控制。

本项目通过 Cloudflare API Token + ACME DNS-01 自动签发证书，并调用宝塔 API 自动部署到指定站点。

核心特点如下：
- 使用 Cloudflare API Token，避免使用高权限 Global API Key。
- 通过宝塔 API 读取站点 SSL 信息并自动更新证书。
- 主进程采用信号触发机制，未触发时 `signal.pause()` 深度休眠，几乎不消耗 CPU。
- 证书更新在子进程执行，支持超时强制回收，避免卡死。
- 验证当前证书是否受信任 CA 签发，异常证书可自动重签。

## 2. 实现原理
脚本运行后会启动守护进程并写入 `update_ssl.pid`。平时不主动执行续签，只有收到 `SIGUSR1` 信号时才触发一次更新流程：

1. 调用宝塔接口获取站点 SSL 信息。
2. 判断是否需要续签：
   - 如果当前证书不受信任，则直接重签。
   - 如果证书受信任，则在到期前 30 天内进入续签窗口。
3. 使用 ACME + Cloudflare DNS-01 签发新证书。
4. 调用宝塔接口部署证书。
5. 恢复休眠，等待下一次信号触发。

## 3. 部署前准备

### 3.1 获取宝塔 API 信息
进入 `宝塔面板 -> 面板设置 -> API接口`，开启 API 后准备以下信息：
- `BT_KEY`：API 密钥。
- `BT_PANEL`：API 地址，建议使用本机地址，例如 `https://127.0.0.1:8888`（按实际端口替换）。

建议将 API 白名单限制为 `127.0.0.1`。

### 3.2 准备 Cloudflare Token
创建 Cloudflare API Token，至少包含：
- `Zone -> DNS -> Edit`
- `Zone -> Zone -> Read`

Token 仅授权目标 Zone，避免过大权限。

可参考教程：
- Cloudflare API Token 获取说明：https://zhuanlan.zhihu.com/p/1918449030331073934

### 3.3 上传项目文件
在宝塔文件管理中创建项目目录，创建并编辑以下文件：
- `update_ssl.py`
- `requirements.txt`

`requirements.txt` 内容如下：

```python
acme==5.3.1
certifi==2026.2.25
dnspython==2.7.0
josepy==2.2.0
pyOpenSSL==25.3.0
python-dotenv==1.2.2
requests==2.32.5
```

`update_ssl.py` 内容如下：
```python
import hashlib
import json
import logging
import logging.handlers
import multiprocessing
import os
import re
import signal
import ssl
import sys
import time
from datetime import datetime, timedelta

import certifi
import dns.resolver
import josepy as jose
import requests
from dotenv import load_dotenv

load_dotenv()
from acme import challenges, client as acme_client, crypto_util, messages
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa
from OpenSSL import crypto as openssl_crypto

# ========================= 日志配置 =========================

LOG_TO_FILE = os.environ.get("LOG_TO_FILE", "false").lower() in ("true", "1", "yes")
LOG_FILE = os.environ.get(
    "LOG_FILE",
    os.path.join(os.path.dirname(os.path.abspath(__file__)), "update_ssl.log"),
)

logger = logging.getLogger("update_ssl")
logger.setLevel(logging.DEBUG)

if LOG_TO_FILE:
    _file_handler = logging.handlers.RotatingFileHandler(
        LOG_FILE, maxBytes=5 * 1024 * 1024, backupCount=1, encoding="utf-8"
    )
    _file_handler.setLevel(logging.DEBUG)
    _file_handler.setFormatter(
        logging.Formatter("%(asctime)s [%(levelname)s] %(message)s", datefmt="%Y-%m-%d %H:%M:%S")
    )
    logger.addHandler(_file_handler)

_console_handler = logging.StreamHandler()
_console_handler.setLevel(logging.INFO)
_console_handler.setFormatter(
    logging.Formatter("[%(levelname)s] %(message)s")
)
logger.addHandler(_console_handler)

# ========================= 配置变量（从环境变量读取） =========================

def _require_env(name):
    """读取必需的环境变量，缺失则报错退出"""
    val = os.environ.get(name)
    if not val:
        logger.error(f"缺少必需的环境变量: {name}")
        sys.exit(1)
    return val

# 宝塔面板配置
BT_KEY = _require_env("BT_KEY")
BT_PANEL = _require_env("BT_PANEL")

# 站点名称（宝塔面板中的站点标识）
SITE_NAME = _require_env("SITE_NAME")

# Cloudflare 配置
CF_API = os.environ.get("CF_API", "https://api.cloudflare.com/client/v4")
CF_API_TOKEN = _require_env("CF_API_TOKEN")
CF_ZONE_NAME = _require_env("CF_ZONE_NAME")
CF_RECORD_NAME = _require_env("CF_RECORD_NAME")

# ACME (Let's Encrypt) 配置
ACME_DIRECTORY_URL = os.environ.get("ACME_DIRECTORY_URL", "https://acme-v02.api.letsencrypt.org/directory")
ACME_EMAIL = os.environ.get("ACME_EMAIL", "test@message.com")

# 子进程超时（秒），默认5分钟
CHILD_TIMEOUT = int(os.environ.get("CHILD_TIMEOUT", "300"))

# 证书续签配置
RENEW_DAYS_BEFORE = 30
DNS_PROPAGATION_TIMEOUT = 120
DNS_CHECK_INTERVAL = 10
DNS_NAMESERVERS = ["223.5.5.5", "8.8.8.8"] # 阿里DNS和Google DNS

# ========================= 宝塔 API 类 =========================


class BtApi:
    def __init__(self, bt_panel, bt_key):
        self.__BT_PANEL = bt_panel
        self.__BT_KEY = bt_key

    def __get_md5(self, s):
        m = hashlib.md5()
        m.update(s.encode("utf-8"))
        return m.hexdigest()

    def __get_key_data(self):
        now_time = int(time.time())
        p_data = {
            "request_token": self.__get_md5(
                str(now_time) + "" + self.__get_md5(self.__BT_KEY)
            ),
            "request_time": now_time,
        }
        return p_data

    def __http_post_cookie(self, url, p_data, timeout=1800):
        import http.cookiejar
        import urllib.request

        cookie_file = "./" + self.__get_md5(self.__BT_PANEL) + ".cookie"
        cookie_obj = http.cookiejar.MozillaCookieJar(cookie_file)
        if os.path.exists(cookie_file):
            cookie_obj.load(cookie_file, ignore_discard=True, ignore_expires=True)

        # 忽略SSL证书验证（宝塔面板自签名证书）
        ctx = ssl.create_default_context()
        ctx.check_hostname = False
        ctx.verify_mode = ssl.CERT_NONE
        handler = urllib.request.HTTPCookieProcessor(cookie_obj)
        https_handler = urllib.request.HTTPSHandler(context=ctx)

        data = urllib.parse.urlencode(p_data).encode("utf-8")
        req = urllib.request.Request(url, data)
        opener = urllib.request.build_opener(handler, https_handler)
        response = opener.open(req, timeout=timeout)
        cookie_obj.save(ignore_discard=True, ignore_expires=True)
        result = response.read()
        if isinstance(result, bytes):
            result = result.decode("utf-8")
        return result

    def get_ssl(self, site_name):
        """获取站点SSL信息"""
        url = self.__BT_PANEL + "/site?action=GetSSL"
        p_data = self.__get_key_data()
        p_data["siteName"] = site_name
        result = self.__http_post_cookie(url, p_data)
        return json.loads(result)

    def set_ssl(self, site_name, key, csr):
        """设置站点SSL证书"""
        url = self.__BT_PANEL + "/site?action=SetSSL"
        p_data = self.__get_key_data()
        p_data["type"] = -1
        p_data["siteName"] = site_name
        p_data["key"] = key
        p_data["csr"] = csr
        result = self.__http_post_cookie(url, p_data)
        return json.loads(result)


# ========================= 证书验证 =========================


def verify_cert_trusted(cert_pem):
    """
    使用 certifi CA 库验证证书是否由权威机构颁发。
    cert_pem 可能包含 fullchain（叶子证书 + 中间证书）。
    返回 True 表示受信任，False 表示不受信任。
    """
    # 加载 CA 证书库
    store = openssl_crypto.X509Store()
    with open(certifi.where(), "r") as f:
        ca_bundle = f.read()
    ca_pems = re.findall(
        r"-----BEGIN CERTIFICATE-----.*?-----END CERTIFICATE-----",
        ca_bundle,
        re.DOTALL,
    )
    for ca_pem in ca_pems:
        try:
            ca_cert = openssl_crypto.load_certificate(
                openssl_crypto.FILETYPE_PEM, ca_pem
            )
            store.add_cert(ca_cert)
        except openssl_crypto.Error:
            pass

    # 从 cert_pem 中提取叶子和中间证书
    all_certs_pem = re.findall(
        r"-----BEGIN CERTIFICATE-----.*?-----END CERTIFICATE-----",
        cert_pem,
        re.DOTALL,
    )
    if not all_certs_pem:
        return False

    # 第一个是叶子证书，其余是中间证书
    leaf_cert = openssl_crypto.load_certificate(
        openssl_crypto.FILETYPE_PEM, all_certs_pem[0]
    )
    intermediate_certs = []
    for ic_pem in all_certs_pem[1:]:
        intermediate_certs.append(
            openssl_crypto.load_certificate(openssl_crypto.FILETYPE_PEM, ic_pem)
        )

    # 验证时提供中间证书链
    store_ctx = openssl_crypto.X509StoreContext(
        store, leaf_cert, intermediate_certs
    )
    try:
        store_ctx.verify_certificate()
        return True
    except openssl_crypto.X509StoreContextError:
        return False


# ========================= Cloudflare DNS =========================


def cf_headers():
    """返回 Cloudflare API 请求头"""
    return {
        "Authorization": f"Bearer {CF_API_TOKEN}",
        "Content-Type": "application/json",
    }


def cf_get_zone_id():
    """获取 Cloudflare Zone ID"""
    resp = requests.get(
        f"{CF_API}/zones",
        params={"name": CF_ZONE_NAME},
        headers=cf_headers(),
    )
    resp.raise_for_status()
    data = resp.json()
    if not data.get("result"):
        raise RuntimeError(f"未找到 Zone: {CF_ZONE_NAME}")
    return data["result"][0]["id"]


def cf_create_txt_record(zone_id, record_name, value):
    """在 Cloudflare 创建 TXT 记录，返回记录 ID"""
    payload = {"type": "TXT", "name": record_name, "content": value, "ttl": 120}
    resp = requests.post(
        f"{CF_API}/zones/{zone_id}/dns_records",
        json=payload,
        headers=cf_headers(),
    )
    resp.raise_for_status()
    result = resp.json()
    if not result.get("success"):
        raise RuntimeError(f"创建 TXT 记录失败: {result.get('errors')}")
    return result["result"]["id"]


def cf_delete_txt_record(zone_id, record_id):
    """删除 Cloudflare TXT 记录"""
    resp = requests.delete(
        f"{CF_API}/zones/{zone_id}/dns_records/{record_id}",
        headers=cf_headers(),
    )
    resp.raise_for_status()


# ========================= ACME 证书签发 =========================


def generate_account_key():
    """每次运行时生成新的 ACME 账户密钥，返回 josepy.JWKRSA"""
    logger.info("生成 ACME 账户密钥...")
    private_key = rsa.generate_private_key(
        public_exponent=65537, key_size=2048, backend=default_backend()
    )
    return jose.JWKRSA(key=private_key)


def wait_for_dns_propagation(record_name, expected_value):
    """等待 DNS TXT 记录传播，使用公共 DNS 服务器检查"""
    logger.info(
        f"等待 DNS 传播: {record_name} -> {expected_value[:20]}..."
    )
    resolver = dns.resolver.Resolver()
    resolver.nameservers = DNS_NAMESERVERS

    start = time.time()
    while time.time() - start < DNS_PROPAGATION_TIMEOUT:
        try:
            answers = resolver.resolve(record_name, "TXT")
            for rdata in answers:
                for txt_string in rdata.strings:
                    if txt_string.decode("utf-8") == expected_value:
                        elapsed = int(time.time() - start)
                        logger.info(f"DNS 传播完成 (耗时 {elapsed}s)")
                        return True
        except (
            dns.resolver.NXDOMAIN,
            dns.resolver.NoAnswer,
            dns.resolver.NoNameservers,
            dns.exception.Timeout,
        ):
            pass
        time.sleep(DNS_CHECK_INTERVAL)

    logger.warning(f"DNS 传播超时 ({DNS_PROPAGATION_TIMEOUT}s)，继续尝试验证...")
    return False


def issue_certificate(domain):
    """
    使用 ACME 协议通过 Cloudflare DNS 验证签发证书。
    返回 (private_key_pem: str, fullchain_pem: str)，失败返回 (None, None)。
    """
    # 1. 加载/创建账户密钥
    account_key = generate_account_key()

    # 2. 创建 ACME 客户端
    logger.info(f"连接 ACME 服务器: {ACME_DIRECTORY_URL}")
    net = acme_client.ClientNetwork(account_key, user_agent="bt-update-ssl/1.0")
    directory = messages.Directory.from_json(
        net.get(ACME_DIRECTORY_URL).json()
    )
    acme = acme_client.ClientV2(directory, net=net)

    # 3. 注册/获取账户
    logger.info(f"注册 ACME 账户 (邮箱: {ACME_EMAIL})...")
    registration = messages.NewRegistration.from_data(
        terms_of_service_agreed=True, email=ACME_EMAIL
    )
    try:
        acme.new_account(registration)
    except Exception as e:
        # 账户可能已存在，尝试获取
        if "already" in str(e).lower() or "conflict" in str(e).lower():
            logger.info("ACME 账户已存在，继续使用")
        else:
            raise
    logger.info("ACME 账户就绪")

    # 4. 生成证书私钥和 CSR
    logger.info(f"为域名 {domain} 生成私钥和 CSR...")
    private_key = rsa.generate_private_key(
        public_exponent=65537, key_size=2048, backend=default_backend()
    )
    private_key_pem = private_key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.TraditionalOpenSSL,
        encryption_algorithm=serialization.NoEncryption(),
    )
    csr_pem = crypto_util.make_csr(private_key_pem, [domain])

    # 5. 创建证书订单
    logger.info("创建证书订单...")
    order = acme.new_order(csr_pem)

    # 6. 处理 DNS-01 验证
    zone_id = cf_get_zone_id()
    created_records = []  # 记录创建的 DNS 记录以便清理

    try:
        for authz in order.authorizations:
            authz_domain = authz.body.identifier.value
            logger.info(f"处理域名验证: {authz_domain}")

            # 查找 DNS-01 挑战
            dns01_chall = None
            for chall_body in authz.body.challenges:
                if isinstance(chall_body.chall, challenges.DNS01):
                    dns01_chall = chall_body
                    break

            if dns01_chall is None:
                raise RuntimeError(
                    f"未找到 DNS-01 验证方式: {authz_domain}"
                )

            # 获取验证值
            response, validation = dns01_chall.response_and_validation(
                account_key
            )
            txt_record_name = dns01_chall.chall.validation_domain_name(
                authz_domain
            )

            # 在 Cloudflare 创建 TXT 记录
            logger.info(
                f"创建 DNS TXT 记录: {txt_record_name} = {validation}"
            )
            record_id = cf_create_txt_record(zone_id, txt_record_name, validation)
            created_records.append(record_id)

            # 等待 DNS 传播
            wait_for_dns_propagation(txt_record_name, validation)

            # 通知 ACME 服务器验证
            logger.info("通知 ACME 服务器进行验证...")
            acme.answer_challenge(dns01_chall, response)

        # 7. 等待并完成订单
        logger.info("等待 ACME 服务器完成验证并签发证书...")
        deadline = datetime.now() + timedelta(seconds=240)
        finalized_order = acme.poll_and_finalize(order, deadline=deadline)

        fullchain_pem = finalized_order.fullchain_pem
        key_str = private_key_pem.decode("utf-8") if isinstance(
            private_key_pem, bytes
        ) else private_key_pem

        logger.info("证书签发成功！")
        return key_str, fullchain_pem

    except Exception as e:
        logger.error(f"证书签发失败: {type(e).__name__}: {e}")
        return None, None

    finally:
        # 清理 DNS 记录
        for record_id in created_records:
            try:
                logger.info(f"清理 DNS TXT 记录: {record_id}")
                cf_delete_txt_record(zone_id, record_id)
            except Exception as e:
                logger.warning(f"清理 DNS 记录失败: {type(e).__name__}: {e}")


# ========================= 证书更新逻辑 =========================


def update_certificate():
    """完整的证书检查与更新流程（在子进程中执行）"""
    try:
        _do_update_certificate()
    except Exception as e:
        logger.error(f"无法处理的异常: {type(e).__name__}: {e}", exc_info=True)


def _do_update_certificate():
    """证书更新的具体实现"""
    bt = BtApi(BT_PANEL, BT_KEY)

    # --- 获取当前SSL信息 ---
    logger.info(f"获取站点 {SITE_NAME} 的SSL信息...")
    try:
        ssl_info = bt.get_ssl(SITE_NAME)
    except Exception as e:
        logger.error(f"获取SSL信息异常: {type(e).__name__}: {e}, 请检查宝塔面板连接和站点名称是否正确")
        return

    # 步骤0: 如果 status 为 false，exit 0
    if not ssl_info.get("status", False):
        logger.info("SSL状态为false，退出")
        return

    # 步骤1: 如果 cert_data 为空，exit 0
    cert_data = ssl_info.get("cert_data")
    if not cert_data:
        logger.info("cert_data为空，退出")
        return

    # 步骤2: 验证当前证书是否由权威机构颁发
    cert_pem = ssl_info.get("csr", "")
    need_renew = False

    if cert_pem:
        try:
            trusted = verify_cert_trusted(cert_pem)
        except Exception as e:
            logger.error(f"证书验证异常: {type(e).__name__}: {e}")
            trusted = False
        if not trusted:
            logger.warning("当前证书不受信任（非权威CA颁发），需要重新签发")
            need_renew = True

    # 步骤3: 如果证书受信任，检查是否需要续签
    if not need_renew:
        not_after_str = cert_data.get("notAfter", "")
        if not_after_str:
            not_after = datetime.strptime(not_after_str, "%Y-%m-%d")
            renew_date = not_after - timedelta(days=RENEW_DAYS_BEFORE)
            today = datetime.now()

            if today < renew_date:
                days_left = (not_after - today).days
                logger.info(
                    f"证书有效期至 {not_after_str}，距到期还有 {days_left} 天，"
                    f"续签阈值为到期前 {RENEW_DAYS_BEFORE} 天，无需续签"
                )
                return
            else:
                logger.info(
                    f"证书将于 {not_after_str} 到期，已进入续签窗口，开始续签"
                )
                need_renew = True

    if not need_renew:
        logger.info("无需续签")
        return

    # 步骤4-5: 使用 ACME 签发证书
    logger.info(f"域名: {CF_RECORD_NAME}")
    key, fullchain = issue_certificate(CF_RECORD_NAME)

    if not key or not fullchain:
        logger.error("证书签发失败")
        return

    # 步骤6: 将证书写入宝塔面板
    logger.info(f"正在将证书部署到站点 {SITE_NAME}...")
    try:
        result = bt.set_ssl(SITE_NAME, key, fullchain)
    except Exception as e:
        logger.error(f"部署证书异常: {type(e).__name__}: {e}")
        return

    if result.get("status"):
        logger.info(f"证书部署成功: {result.get('msg', '')}")
    else:
        logger.error(f"证书部署失败: {result.get('msg', '未知错误')}")


# ========================= 守护进程主逻辑 =========================


def main():
    child_proc = None
    should_update = False

    def handle_sigusr1(signum, frame):
        nonlocal should_update
        should_update = True

    def handle_sigterm(signum, frame):
        """收到 SIGTERM/SIGINT 时退出"""
        nonlocal child_proc
        logger.info("收到退出信号，正在退出...")
        if child_proc and child_proc.is_alive():
            child_proc.terminate()
            child_proc.join(timeout=10)
            if child_proc.is_alive():
                child_proc.kill()
        # 删除 PID 文件
        if os.path.exists(pid_file):
            os.unlink(pid_file)
        sys.exit(0)

    # 注册信号
    signal.signal(signal.SIGUSR1, handle_sigusr1)
    signal.signal(signal.SIGTERM, handle_sigterm)
    signal.signal(signal.SIGINT, handle_sigterm)

    # 写入 PID 文件
    pid_file = os.path.join(os.path.dirname(os.path.abspath(__file__)), "update_ssl.pid")
    with open(pid_file, "w") as f:
        f.write(str(os.getpid()))

    logger.info("=" * 30)
    logger.info(f"守护进程已启动，PID: {os.getpid()}")
    logger.info(f"启动时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    logger.info(f"PID 文件: {pid_file}")
    logger.info(f"日志文件: {LOG_FILE}")
    logger.info(f"发送 SIGUSR1 触发证书更新: kill -USR1 $(cat {pid_file})")
    logger.info(f"子进程超时: {CHILD_TIMEOUT}s")

    while True:
        should_update = False

        # 休眠，等待系统中断信号唤醒
        signal.pause()

        if not should_update:
            continue

        # 启动子进程执行更新
        logger.info("收到 SIGUSR1，启动子进程执行证书更新...")
        child_proc = multiprocessing.Process(
            target=update_certificate, daemon=True
        )
        child_proc.start()

        # 等待子进程完成（期间忽略 SIGUSR1）
        child_proc.join(timeout=CHILD_TIMEOUT)

        if child_proc.is_alive():
            logger.warning(f"子进程超时 ({CHILD_TIMEOUT}s)，强制终止")
            child_proc.terminate()
            child_proc.join(timeout=5)
            if child_proc.is_alive():
                child_proc.kill()
                child_proc.join()

        exit_code = child_proc.exitcode
        if exit_code == 0:
            logger.info("子进程正常完成")
        else:
            logger.warning(f"子进程退出码: {exit_code}")

        child_proc = None
        should_update = False
        logger.info("恢复休眠，等待下次信号...")


if __name__ == "__main__":
    main()

```

### 3.4 准备 `SITE_NAME`
`SITE_NAME` 必须与宝塔 API 中站点标识一致，不等同于域名。

推荐获取方式：
1. 打开目标站点 SSL 设置页面。
2. 按 `F12` 打开开发者工具，切换到“网络”。
3. 在页面点击一次 `SSL`，观察请求 `/site?action=GetSSL` 中 `负载` 的 `siteName`。
4. 将该值填入环境变量 `SITE_NAME`。

## 4. 在宝塔中部署 Python 服务

### 4.1 创建 Python 项目
进入 `网站 -> Python项目 -> 添加项目`，建议配置：
- 项目路径：`update_ssl.py` 所在目录。
- Python 版本：建议 `3.13.3`（若无可先安装）。
- 虚拟环境：新建即可，名称自定义。
- 启动命令：`python3 update_ssl.py`
- 启动用户：建议使用普通用户；使用 `www` 也可运行。
- 安装依赖：选择 `requirements.txt`。

### 4.2 配置环境变量
在 Python 项目环境变量中配置（见第 8 章完整说明）。最少需要填写：

```python
BT_KEY=你的宝塔API密钥
BT_PANEL=https://127.0.0.1:你的宝塔端口
SITE_NAME=宝塔站点标识siteName
CF_API_TOKEN=Cloudflare API Token
CF_ZONE_NAME=主域名，例如example.com
CF_RECORD_NAME=签发证书的完整域名，例如www.example.com
```

### 4.3 启动项目
启动 Python 项目后，脚本会写入 PID 文件：

```python
<项目目录>/update_ssl.pid
```

后续计划任务通过该 PID 文件向主进程发送 `SIGUSR1` 信号。

### 4.4 项目日志配置
进入 `Python项目 -> 设置 -> 项目日志`，建议将日志目录设在项目目录下 `logs/`。

说明：
- 默认日志输出到控制台（可在宝塔项目日志查看）。
- 若需要额外文件日志，请设置 `LOG_TO_FILE=true` 并配置 `LOG_FILE`（见第 8 章）。

## 5. 初始化 SSL 状态（建议）
脚本在读取宝塔 SSL 信息时，若 `status=false` 或 `cert_data` 为空会直接退出本次更新。

因此建议先在宝塔给站点配置一张临时证书（可自签），确保 SSL 状态已启用且存在证书数据，然后再交由脚本自动续签替换。

可选参考（自签证书生成工具）：
- top.tools：https://tools.top/certificate-generate.html

## 6. 配置计划任务触发更新

### 6.1 新建计划任务
进入 `计划任务 -> 添加任务 -> Shell脚本`，脚本内容如下：

```python
kill -USR1 $(cat /path/to/your/project/update_ssl.pid)
```

将 `/path/to/your/project` 替换为实际项目路径。

建议：
- 任务执行用户与 Python 项目启动用户保持一致。
- 执行周期按需求设置（例如每天一次）。

### 6.2 手动执行一次
保存任务后先手动执行一次，确认触发链路正常。

## 7. 验证部署是否成功
在 `网站 -> Python项目 -> 你的项目 -> 设置 -> 项目日志` 观察日志。

关键日志示例：
- `收到 SIGUSR1，启动子进程执行证书更新...`
- `证书签发成功！`
- `证书部署成功: ...`
- `恢复休眠，等待下次信号...`

同时到站点 SSL 页面确认新证书是否已自动部署。

## 8. 环境变量说明（完整）

### 8.1 必填变量

| 变量名 | 说明 | 示例 |
| --- | --- | --- |
| `BT_KEY` | 宝塔 API 密钥 | `xxxx` |
| `BT_PANEL` | 宝塔 API 地址 | `https://127.0.0.1:8888` |
| `SITE_NAME` | 宝塔站点标识（`siteName`） | `example.com` 或面板内部标识 |
| `CF_API_TOKEN` | Cloudflare API Token | `xxxx` |
| `CF_ZONE_NAME` | Cloudflare Zone 名称 | `example.com` |
| `CF_RECORD_NAME` | 需要签发证书的完整域名 | `www.example.com` |

### 8.2 选填变量

| 变量名 | 默认值 | 说明 |
| --- | --- | --- |
| `CF_API` | `https://api.cloudflare.com/client/v4` | Cloudflare API 基础地址 |
| `ACME_DIRECTORY_URL` | `https://acme-v02.api.letsencrypt.org/directory` | ACME 目录地址（默认生产环境） |
| `ACME_EMAIL` | `test@message.com` | ACME 账户邮箱 |
| `CHILD_TIMEOUT` | `300` | 子进程超时时间（秒） |
| `LOG_TO_FILE` | `false` | 是否写入文件日志（`true/1/yes` 生效） |
| `LOG_FILE` | `<项目目录>/update_ssl.log` | 文件日志路径 |

### 8.3 固定参数（需改代码）
以下参数当前写死在脚本中，如需调整需修改代码：
- 续签窗口：`RENEW_DAYS_BEFORE = 30`
- DNS 传播超时：`DNS_PROPAGATION_TIMEOUT = 120`
- DNS 查询间隔：`DNS_CHECK_INTERVAL = 10`
- DNS 解析服务器：`223.5.5.5`、`8.8.8.8`

## 9. 常见问题与排查

### 9.1 定时任务执行但无更新日志
- 检查 `update_ssl.pid` 是否存在。
- 检查 PID 是否对应运行中的 Python 进程。
- 检查任务用户是否有权限读取 PID 文件并发送信号。

### 9.2 日志提示缺少环境变量
脚本启动时会校验必填变量，缺失即退出。请逐项核对第 8.1 节。

### 9.3 Cloudflare 相关报错
- 检查 Token 权限是否包含 `DNS Edit` 与 `Zone Read`。
- 检查 `CF_ZONE_NAME` 与实际 Zone 是否一致。
- 检查 `CF_RECORD_NAME` 是否属于该 Zone。

### 9.4 ACME 验证失败
- 检查 DNS 记录是否成功创建。
- 检查域名是否正确解析到当前站点。
- 如 DNS 生效慢，可适当增大 `DNS_PROPAGATION_TIMEOUT`（改代码）。

### 9.5 宝塔部署失败
- 检查 `BT_PANEL` 地址和端口。
- 检查 `BT_KEY` 是否正确。
- 检查 `SITE_NAME` 是否准确。