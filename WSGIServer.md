# First_rep

import socket
import multiprocessing
import re
import dynamic.mini_web


class WSIGServer(object):

    def __init__(self):
         # 创建套接字
         server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
         # 设置套接字选项
         server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
         # 绑定固定端口
         server_socket.bind(("", 8080))
         # 设置为被动套接字
         server_socket.listen(128)
         # 设置套接字在函数间可进行传递
         self.server_socket = server_socket

    def request_handler(self, client_socket):
        # 接收数据
        recv_data =  client_socket.recv(4096)
        if not recv_data:
            print("客户端已经断开链接")
            client_socket.close()
            return
        # 解码
        recv_str_data = recv_data.decode()
        # 通过正则提取
        # 格式：GET /index.html HTTP/1.1
        ret = re.match(r"[^/]+([^ ]+)", recv_str_data)
        if ret:
            path_info = ret.group(1)
        else:
            path_info = "/"
        print("用户请求路径是%s" % path_info)

        # 通过正则提取url，发现/意味访问的是主页
        # 一般主页是/index.html
        if path_info == "/":
            path_info = "/index.html"
         # 判断请求静态还是动态
        if not path_info.endswith(".html"):  # 不是以html结尾--->即静态       
            try:
                with open("./html" + path_info, "rb") as f:
                    file_data = f.read()
            except Exception as e:
                # 请求失败 响应行 响应头 空行 响应体
                response_header = "HTTP/1.1 404 Not Found\r\n"
                response_header += "Server: pythonweb1.1\r\n"
                response_header += "\r\n"
                response_body = "ERROR!!!"
                response_data = response_header + response_body
                client_socket.send(response_data.encode())

            else:
                # 给客户端回复HTTP响应报文：响应行 + 响应头 +空行 + 响应体
                # request---->请求
                # response --->应答（响应）
                # 响应头(response_header)
                response_header = "HTTP/1.1 200 OK\r\n"
                response_header += "Server: pythonWeb1.1\r\n"
                response_header += "\r\n"
                # 响应体(response_body)
                response_body = file_data
                # 拼接报文
                response = response_header.encode("utf-8") + response_body
                # 发送
                client_socket.send(response)
            finally:
                client_socket.close()
        else:
            # 以.html 结尾
            """            
            response_header = "HTTP/1.1 200 OK\r\n"
            response_header += "Server: pythonWeb1.1\r\n"
            response_header += "Content-Type: text/html; charset=utf-8\r\n"
            response_header += "\r\n"
           """


           # 响应体
            env = dict()
            env["PATH_INFO"] = path_info
            response_body = dynamic.mini_web.application(env, self.set_headers)
            """
            if path_info == "/index.py":
                response_body = mini_web.index()
            elif path_info == "/center.py":
                response_body = mini_web.center()
            else:
                response_body = "该页面不存在。。。"
            """
            response = self.response_header + response_body
            # 发送
            client_socket.send(response.encode())

    def set_headers(self, status, headers):
        print("---被调用---")
        response_header = "HTTP/1.1 %s\r\n" % status
        for temp in headers:
            response_header += "%s: %s\r\n" % (temp[0], temp[1])
        response_header += "\r\n"
        self.response_header = response_header

    def run(self):
        """等待客户端的链接，然后创建子进程为其服务"""
        while True:
            client_socket, client_addr = self.server_socket.accept()
            p = multiprocessing.Process(target=self.request_handler, args=(client_socket,))
            p.start()

            client_socket.close()
def main():
    # 创建一个实例对象
    wsgi_server = WSIGServer()
    # 调用对象的方法运行
    wsgi_server.run()

if __name__ == "__main__":
    main()
