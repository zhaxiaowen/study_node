### paramiko

```
# paramiko

# -*- coding: utf-8 -*-
# !/usr/bin/env python
# Software: PyCharm
# __author__ == "YU HAIPENG"
# fileName: sshproxy.py
# Month: 九月
# time: 2020/9/26 20:43
# noqa
"""
# 对于更多限制命令，需要在系统中设置
/etc/sudoers

Defaults    requiretty
Defaults:cmdb    !requiretty

"""
import os
import platform
import paramiko
import re
from re import compile

__all__ = ["SSHProxy", "get_sys", "get_home"]

split_reg = compile(r'[\\|/]')

class SSHProxy(object):

    def __init__(
            self,
            hostname,
            username,
            port=22,
            private_key_path=None,
            password=None):
        self.hostname = hostname
        self.port = port
        self.username = username
        self.private_key_path = main_path(private_key_path)
        self.password = password
        self.transport = None
        self.__ssh = None

    def open(self):
        """ 初始化连接 """
        self.transport = paramiko.Transport((self.hostname, self.port))
        if self.private_key_path:
            private_key = paramiko.RSAKey.from_private_key_file(
                self.private_key_path)
            try:
                self.transport.connect(username=self.username, pkey=private_key)
            except paramiko.ssh_exception.AuthenticationException as e:
                if self.password:
                    self.transport.auth_password(username=self.username, password=self.password)
                else:
                    raise e
        else:
            if self.password:
                self.transport.connect(username=self.username, password=self.password)
            else:
                id_rsa = os.path.join(get_home(), f".ssh", "id_rsa")
                if os.path.isfile(id_rsa):
                    private_key = paramiko.RSAKey.from_private_key_file(id_rsa)
                    self.transport.connect(username=self.username, pkey=private_key)
                    return
                raise paramiko.ssh_exception.AuthenticationException("authentication failed")

    def close(self):
        self.transport.close()
        self.transport = None
        self.__ssh = None

    def command(self,
                cmd,
                buf_size=-1,
                timeout=None,
                get_pty=False,
                environment=None,
                exec_source=False,
                ):
        if self.__ssh is None:
            ssh = paramiko.SSHClient()
            """
            AutoAddPolicy 自动添加主机名及主机密钥到本地 HostKeys 对象 , 不依赖 load_system_host_key 的配置 .
                即新建立 ssh 连接时不需要再输入 yes 或 no 进行确认
                1. AutoAddPolicy 自动添加主机名及主机密钥到本地 HostKeys 对象，不依赖 load_system_host_key 的配置。
                    即新建立 ssh 连接时不需要再输入 yes 或 no 进行确认
                2.WarningPolicy 用于记录一个未知的主机密钥的 python 警告。并接受，功能上和 AutoAddPolicy 类似，但是会提示是新连接
                3.RejectPolicy 自动拒绝未知的主机名和密钥，依赖 load_system_host_key 的配置。此为默认选项
            """
            ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
            ssh._transport = self.transport
            self.__ssh = ssh
        if exec_source:
            cmd = f"source /etc/profile;{cmd}"
        stdin, stdout, stderr = self.__ssh.exec_command(
            cmd, bufsize=buf_size,
            timeout=timeout,
            get_pty=get_pty,
            environment=environment)
        stdout_str = self.bytes_to_string(stdout.read())
        stderr_str = self.bytes_to_string(stderr.read())
        # ssh.close()
        return {"stdout": stdout_str, "stderr": stderr_str}

    @staticmethod
    def bytes_to_string(bytes_info):
        if isinstance(bytes_info, bytes):
            return str(bytes_info, 'utf8')
        return bytes_info

    def upload(self, local_path, remote_path, callback=None, confirm=True):
        """
        上传文件
        @param local_path:
        @param remote_path:
        @param callback: 可选回调函数（形式：``func（int，int）```），接受
                        到目前为止传输的字节数和要传输的总字节数
        @param confirm: 以后是否对文件执行 stat（）来确认文件
        @return:
        """
        sftp = paramiko.SFTPClient.from_transport(self.transport)
        try:
            sftp.put(main_path(local_path), remote_path, callback, confirm)
        finally:
            sftp.close()

    def download(self, remote_path, local_path, callback=None):
        """

        @param remote_path:
        @param local_path:
        @param callback: 可选回调函数（形式：``func（int，int）```），接受
                        到目前为止传输的字节数和要传输的总字节数
        @return:
        """
        sftp = paramiko.SFTPClient.from_transport(self.transport)
        try:
            sftp.get(remote_path, main_path(local_path), callback)
        finally:
            sftp.close()

    def __enter__(self):
        self.open()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()

def main_path(path: str):
    """
    路径总函数
    @param path:
    @return:
    """
    if path is None:
        return path
    current_path = os.getcwd()
    if path.startswith('..'):
        path = _wne_path(_parse(path, current_path))
    elif path.startswith('.'):
        path = _wne_path(path[1:], current_path)
    elif path:
        path = _wne_path(path)
    else:
        raise ValueError(' 文件路径错误 ')
    return path

def _parse(path: str, current_path):
    """
    解析 .. 路径
    :param path:
    :param current_path:
    :return:
    """
    new_path_args = list(filter(lambda x: x != '', _get_path_params(path)))
    row = 0
    while row < len(new_path_args):
        if new_path_args[row] == '..':
            current_path = os.path.dirname(current_path)
            new_path_args.remove(new_path_args[row])
            row -= 1
        else:
            break
        row += 1
    return os.path.join(current_path, *new_path_args)

def _wne_path(new_path: str, current_path=None):
    new_path_args = _get_path_params(new_path)
    if current_path:
        path = os.path.join(current_path, *new_path_args)
    else:
        sys_str = get_sys()
        if sys_str == "Windows":
            if new_path_args[0].find(':') != -1:
                new_path_args[0] += os.sep
            path = os.path.join(*new_path_args)
        elif sys_str in ["Linux", "Mac", "Darwin"]:
            if new_path_args[0] == '':
                new_path_args[0] = os.sep
            path = os.path.join(*new_path_args)
        else:
            path = new_path
    return path

def get_sys():
    """
    平台
    @return:
    """
    sys_str = platform.system()
    return sys_str

def get_home(path='~'):
    """
    家目录
    @return:
    """
    return os.path.expanduser(path)

def _get_path_params(path):
    """
    路径参数
    @param path:
    @return:
    """

    return split_reg.split(path)

def get_message():
    node_list = []
    jmv_list = []
    server_map = []
    port_map = []
    for message in all:
        with SSHProxy(
                hostname=message["ip"],
                username="carapp",
                # password="39ca04fbf62d"
                private_key_path="/home/carapp/.ssh/id_rsa"
        ) as ssh:
            res = ssh.command("hostname", )
            hostname = res['stdout'].replace("\n", "")
        name = re.split(r"dc2-tc-gz[0-9]-", hostname)
        if not name:
            break
        name = name[1].split("-")
        # name = hostname.split("dc2-tc-gz6-")[1].split("-")
        if len(name) == 3:
            service_group_name = "{0}-{1}-app".format(name[1], name[0])
            product_line = name[1]
            unique_jvm_id = "{0}-{1}".format(name[1], name[2])
            unique_node_id = "{0}-{1}-{2}".format(name[1], name[0], name[2])
        elif len(name) == 4:
            service_group_name = "{0}-{1}-{2}-app".format(name[2], name[0], name[1])
            product_line = name[2]
            unique_jvm_id = "{0}-{1}-{2}".format(name[2], name[1], name[3])
            unique_node_id = "{0}-{1}-{2}-{3}".format(name[2], name[0], name[1], name[3])
        else:
            return
        service_type = "app"
        node_result = "{0}   {1}   {2}   {3}  {4}   {5}  59100  node_exporter  {6}".format(service_group_name,
                                                                                           product_line,
                                                                                           service_type,
                                                                                           unique_node_id,
                                                                                           message["ip"],
                                                                                           hostname,
                                                                                           message["service_name"])
        jvm_result = "{0}   {1}   {2}   {3}  {4}  {5}  58000  jmx_exporter  {6}".format(service_group_name,
                                                                                        product_line,
                                                                                        service_type,
                                                                                        unique_jvm_id,
                                                                                        message["ip"],
                                                                                        hostname,
                                                                                        message["service_name"])
        server_result = """                      "{0}": ["{1}"],""".format(hostname, message["service_name"])
        port_result = """            "{0}": "{1}",""".format(message["service_name"], message["port"])
        node_list.append(node_result)
        jmv_list.append(jvm_result)
        server_map.append(server_result)
        port_map.append(port_result)
    port_map = list(set(port_map))
    for i in node_list:
        print(i)
    for j in jmv_list:
        print(j)
    for x in server_map:
        print(x)
    for y in port_map:
        print(y)

def delete_monitor():
    for message in all:
        if message["product_line"] == "wyc":
            monitor_server_ip = prometheus_base_monitor_ip[1]["wyc"]
        else:
            monitor_server_ip = prometheus_base_monitor_ip[0]["other"]
        with SSHProxy(
                hostname=monitor_server_ip,
                username="carapp",
                # password="39ca04fbf62d"
                private_key_path="/home/carapp/.ssh/id_rsa"
        ) as ssh:
            res = ssh.command(
                "curl --request PUT http://127.0.0.1:8500/v1/agent/service/deregister/{0}".format(message["ip"]), )
            print(res['stdout'].replace("\n", ""))

if __name__ == '__main__':
    all = [
        {"ip": "172.19.81.228", "service_name": "oaWorkspace", "product_line": "wyc", "port": 18940},
        {"ip": "172.19.81.229", "service_name": "oaWorkspace", "product_line": "wyc", "port": 18940},
        {"ip": "172.19.81.230", "service_name": "flowServiceConsoleImpl", "product_line": "wyc", "port": 18940},
        {"ip": "172.19.81.231", "service_name": "flowServiceConsoleImpl", "product_line": "wyc", "port": 18940},
        {"ip": "172.19.81.232", "service_name": "flowServiceImpl", "product_line": "wyc", "port": 18940},
        {"ip": "172.19.81.233", "service_name": "flowServiceImpl", "product_line": "wyc", "port": 18940},
    ]
    prometheus_base_monitor_ip = [
        {"other": "172.25.2.5"},
        {"wyc": "172.25.2.7"}
    ]
    get_message()
    for message in all:
        with SSHProxy(
                hostname=message["ip"],
                # password="39ca04fbf62d"
                private_key_path="/home/carapp/.ssh/id_rsa"
        ) as ssh:
            res = ssh.command("hostname", )
            hostname = res['stdout'].replace("\n", "")

```

