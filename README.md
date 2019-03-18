# Pty-Qt - C++ library for work with PseudoTerminals

Pty-Qt is small library for access to console applications by pseudo-terminal interface on Mac, Linux and Windows. On Mac and Linux you can use standard PseudoTerminal API and on Windows you can use WinPty or ConPty.

### CI Status

Ubuntu / MacOS X: [![Build Status](https://travis-ci.org/kafeg/ptyqt.svg?branch=master)](https://travis-ci.org/kafeg/ptyqt)

Windows: [![Build Status](https://ci.appveyor.com/api/projects/status/github/kafeg/ptyqt?svg=true)](https://ci.appveyor.com/project/kafeg/ptyqt)

## Pre-Requirements and build
  - ConPty part works only on Windows 10 >= 1903 (build > 18309) and can be built only with Windows SDK >= 10.0.18346.0 (maybe >= 17134, but not sure)
  - WinPty part requires winpty sdk for build and winpty.dll with winpty-agent.exe for deployment with target application. WinPty can work on Windows XP and later (depended on used build SDK: vc140 / vc140_xp). You can't link WinPty libraries inside your App, because it use cygwin for build.
  - UnixPty part can work on both Linux/Mac versions, because it based on standard POSIX pseudo terminals API
  - Еarget platforms: x86 or x64
  - Required Qt >= 5.10
  - !!!IMPORTANT!!! Now library has some problems with ConPty, work in progress: https://github.com/Microsoft/console/issues/388

### Build on Windows (Git Bash)
```sh
git clone https://github.com/Microsoft/vcpkg.git vcpkg
cd vcpkg
./bootstrap-vcpkg.sh
./vcpkg.exe integrate install
./vcpkg.exe install qt-base:x64-windows qt5-network:x64-windows
export VCPKG_ROOT=`pwd`
cd ..
git clone https://github.com/kafeg/ptyqt.git ptyqt
mkdir ptyqt-build
cd ptyqt-build
${VCPKG_ROOT}/downloads/tools/cmake-3.12.4-windows/cmake-3.12.4-win32-x86/bin/cmake.exe ../ptyqt "-DCMAKE_TOOLCHAIN_FILE=${VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake" "-DVCPKG_TARGET_TRIPLET=x64-windows"
${VCPKG_ROOT}/downloads/tools/cmake-3.12.4-windows/cmake-3.12.4-win32-x86/bin/cmake.exe --build . --target winpty
${VCPKG_ROOT}/downloads/tools/cmake-3.12.4-windows/cmake-3.12.4-win32-x86/bin/cmake.exe --build .
```

### Build on Ubuntu
```sh
sudo apt-get install qtbase5-dev cmake libqt5websockets5-dev
git clone https://github.com/kafeg/ptyqt.git ptyqt
mkdir ptyqt-build
cd ptyqt-build
cmake ../ptyqt
cmake --build .
```

### Build on Mac
```sh
brew install qt cmake
git clone https://github.com/kafeg/ptyqt.git ptyqt
mkdir ptyqt-build
cd ptyqt-build
cmake ../ptyqt "-DCMAKE_PREFIX_PATH=/usr/local/Cellar/qt/5.12.1"
cmake --build .
```

## Usage
Standard way: build and install library then link it to your project and check examples for sample code. Simple code snippet from 'xtermjs' example:
```cpp
#include <QCoreApplication>
#include <QWebSocketServer>
#include <QWebSocket>
#include "ptyqt.h"
#include <QTimer>
#include <QProcessEnvironment>

#define PORT 4242

int main(int argc, char *argv[])
{
    QCoreApplication app(argc, argv);

    //start WebSockets server for receive connections from xterm.js
    QWebSocketServer wsServer("TestServer", QWebSocketServer::NonSecureMode);
    if (!wsServer.listen(QHostAddress::Any, PORT))
        return 1;

    QMap<QWebSocket *, IPtyProcess *> sessions;

    //create new session on new connection
    QObject::connect(&wsServer, &QWebSocketServer::newConnection, [&wsServer, &sessions]()
    {
        //handle new connection
        QWebSocket *wSocket = wsServer.nextPendingConnection();

        //use cmd.exe or bash, depends on target platform
        IPtyProcess::PtyType ptyType = IPtyProcess::WinPty;
        QString shellPath = "c:\\Windows\\system32\\cmd.exe";
#ifdef Q_OS_UNIX
        shellPath = "/bin/bash";
        ptyType = IPtyProcess::UnixPty;
#endif

        //create new Pty instance
        IPtyProcess *pty = PtyQt::createPtyProcess(ptyType);

        qDebug() << "New connection" << wSocket->peerAddress() << wSocket->peerPort() << pty->pid();

        //start Pty process ()
        pty->startProcess(shellPath, QProcessEnvironment::systemEnvironment().toStringList(), 80, 24);

        //connect read channel from Pty process to write channel on websocket
        QObject::connect(pty->notifier(), &QIODevice::readyRead, [wSocket, pty]()
        {
            wSocket->sendTextMessage(pty->readAll());
        });

        //connect read channel of Websocket to write channel of Pty process
        QObject::connect(wSocket, &QWebSocket::textMessageReceived, [wSocket, pty](const QString &message)
        {
            pty->write(message.toUtf8());
        });

        //...
        //for example handle disconnections, process crashes and stuff like that...
        //...

        //add connection to list of active connections
        sessions.insert(wSocket, pty);
    });

    //stop eventloop if needed
    //QTimer::singleShot(5000, [](){ qApp->quit(); });

    //exec eventloop
    bool res = app.exec();

    QMapIterator<QWebSocket *, IPtyProcess *> it(sessions);
    while (it.hasNext())
    {
        it.next();

        it.key()->deleteLater();
        delete it.value();
    }
    sessions.clear();

    return res;
}
```

## Examples
### XtermJS
- build and run example
- install nodejs (your prefer way)
- open console and run:
```sh
cd ptyqt/examples/xtermjs
npm install xterm
npm install http-server -g
http-server ./
```
- open http://127.0.0.1:8080/ in Web browser

### QTerminal
in dev...

## More information
Resources used to develop this library:
  - https://github.com/Microsoft/node-pty
  - https://github.com/Microsoft/console
  - https://github.com/rprichard/winpty
  - https://github.com/xtermjs/xterm.js
  - https://github.com/lxqt/qterminal
  - https://devblogs.microsoft.com/commandline/windows-command-line-introducing-the-windows-pseudo-console-conpty/
  - https://devblogs.microsoft.com/commandline/windows-command-line-backgrounder/
