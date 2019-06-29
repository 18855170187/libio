# libio
A  simple io interface for linux network programing, containing multiple IO model.

Linuxƽ̨��IO�����,��װ����ioģ��,����reactor,���߳�reactor,�����reactor(master-worker),proactorģ��.

## �ص�

���ö��ֻ������,����������,��ʱ��,����io��·���û���,�̳߳�,�źŹ���

## example

�� echo serverΪ��, ��װ����IOģ��

__1. ���߳�reactorģ��__
```c++
#include <core/epoll.hh>
#include <model/thread_loop.hh>

void test_thread_loop() {
    wxg::thread_loop server;

    server.resize(4);

    server.set_read_handler([](int fd) {
        wxg::buffer buf;
        int res = buf.read(fd);

        if (res > 0) {
            cout << buf.get() << endl;
            buf.write(fd);
        } else {
            close(fd);
        }
    });

    server.start("127.0.0.1", 8081);
}
```

__2. �����reactorģ�� (master-worker����ģʽ)__

��nginxʵ��master/workerģ��
```c++
#include <core/epoll.hh>
#include <model/process_loop.hh>

void test_process_loop() {
    wxg::process_loop server;

    server.resize(4);

    server.set_read_handler([](int fd) {
        wxg::buffer buf;
        int res = buf.read(fd);

        if (res > 0) {
            cout << buf.get() << endl;
            buf.write(fd);
        } else {
            close(fd);
        }
    });

    server.start("127.0.0.1", 8081);
}
```

__3. proactorģ��__

��asio��ʵ��proactorģ��,����reactorģ�ͽ����̳߳�ģ��proactor�첽io.

```c++
#include <core/epoll.hh>
#include <model/proactor.hh>

class session : public std::enable_shared_from_this<session> {
   private:
    int socket = -1;
    wxg::buffer buf;
    wxg::proactor<wxg::epoll> *io_context;

   public:
    session(int socket, wxg::proactor<wxg::epoll> *io_context)
        : socket(socket), io_context(io_context) {}

    void do_read() {
        auto self(shared_from_this());
        io_context->async_read(socket, [this, self]() {
            int res = buf.read(socket);
            if (res > 0) do_write();
        });
    }

    void do_write() {
        auto self(shared_from_this());
        io_context->async_write(socket, [this, self]() {
            int res = buf.write(socket);
            if (res >= 0) do_read();
        });
    }
};

void do_accept(int socket, wxg::proactor<wxg::epoll> *io_context) {
    io_context->async_accept(socket, [socket, io_context]() {
        int fd = wxg::tcp::accept(socket);
        cout << "socket accept fd=" << fd << endl;

        if (fd > 0) {
            std::make_shared<session>(fd, io_context)->do_read();
        }

        do_accept(socket, io_context);
    });
}

void test_proactor() {
    wxg::proactor<wxg::epoll> io_context;

    int socket = wxg::tcp::get_nonblock_socket();
    wxg::tcp::bind(socket, "127.0.0.1", 8081);
    wxg::tcp::listen(socket);

    do_accept(socket, &io_context);

    std::thread t([&io_context]() { io_context.run(); });

    sleep(10);

    io_context.run();
}
```