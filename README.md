![](./images/ecapture-logo-400x400.png)

[中文介绍](./README_CN.md) | English

[![GitHub stars](https://img.shields.io/github/stars/ehids/ecapture.svg?label=Stars&logo=github)](https://github.com/ehids/ecapture)
[![GitHub forks](https://img.shields.io/github/forks/ehids/ecapture?label=Forks&logo=github)](https://github.com/ehids/ecapture)
[![CI](https://github.com/ehids/ecapture/actions/workflows/codeql-analysis.yml/badge.svg)](https://github.com/ehids/ecapture/actions/workflows/code-analysis.yml)
[![Github Version](https://img.shields.io/github/v/release/ehids/ecapture?display_name=tag&include_prereleases&sort=semver)](https://github.com/ehids/ecapture/releases)

### eCapture(旁观者):  capture SSL/TLS text content without CA cert Using eBPF.

> **Note**
>
> Support Linux Kernel 4.15 or newer,Support Android Kernel 5.4 or newer.
>
> Do not support Windows and macOS system.
----

#  How eCapture works

![](./images/how-ecapture-works.png)

* SSL/TLS text context capture, support openssl\libressl\boringssl\gnutls\nspr(nss) libraries.
* bash audit, capture bash command for Host Security Audit.
* mysql query SQL audit, support mysqld 5.6\5.7\8.0, and mariadDB.

# eCapture Architecture
![](./images/ecapture-architecture.png)

# eCapture User Manual

[![eCapture User Manual](./images/ecapture-user-manual.png)](https://www.youtube.com/watch?v=CoDIjEQCvvA "eCapture User Manual")

# Getting started

## use ELF binary file

Download ELF zip file [release](https://github.com/ehids/ecapture/releases) , unzip and use by
command `./ecapture --help`.

* Linux kernel version >= 4.15 is required.
* Enable BTF [BPF Type Format (BTF)](https://www.kernel.org/doc/html/latest/bpf/btf.html)  (Optional, 2022-04-17)

## Command line options

> **Note**
>
> Need ROOT permission.
>
eCapture search `/etc/ld.so.conf` file default, to search load directories of  `SO` file, and search `openssl` shard
libraries location. or you can use `--libssl`
flag to set shard library path.

If target program is compile statically, you can set program path as `--libssl` flag value directly。

### Pcapng result

`./ecapture tls -i eth0 -w pcapng -p 443` capture plaintext packets save as pcapng file, use `Wireshark` read it
directly.

### plaintext result

`./ecapture tls` will capture all plaintext context ,output to console, and capture `Master Secret` of `openssl TLS`
save to `ecapture_master.log`. You can also use `tcpdump` to capture raw packet,and use `Wireshark` to read them
with `Master Secret` settings.

>

### check your server BTF config：

```shell
cfc4n@vm-server:~$# uname -r
4.18.0-305.3.1.el8.x86_64
cfc4n@vm-server:~$# cat /boot/config-`uname -r` | grep CONFIG_DEBUG_INFO_BTF
CONFIG_DEBUG_INFO_BTF=y
```

### tls command

capture tls text context.
Step 1:
```shell
./ecapture tls --hex
```

Step 2:
```shell
curl https://github.com
```

### libressl&boringssl
```shell
# for installed libressl, libssl.so.52 is the dynamic ssl lib
vm@vm-server:~$ ldd /usr/local/bin/openssl
	linux-vdso.so.1 (0x00007ffc82985000)
	libssl.so.52 => /usr/local/lib/libssl.so.52 (0x00007f1730f9f000)
	libcrypto.so.49 => /usr/local/lib/libcrypto.so.49 (0x00007f1730d8a000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f1730b62000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f17310b2000)

# use the libssl to config the libssl.so path
vm@vm-server:~$ sudo ./ecapture tls --libssl="/usr/local/lib/libssl.so.52" --hex

# in another terminal, use the command, then type some string, watch the output of ecapture
vm@vm-server:~$ /usr/local/bin/openssl s_client -connect github.com:443

# for installed boringssl, usage is the same
/path/to/bin/bssl s_client -connect github.com:443
```

### bash command
capture bash command.
```shell
ps -ef | grep foo
```

# What's eBPF
[eBPF](https://ebpf.io)

## uprobe HOOK

### openssl\libressl\boringssl hook
eCapture hook`SSL_write` \ `SSL_read` function of shared library `/lib/x86_64-linux-gnu/libssl.so.1.1`. get text context, and send message to user space by [eBPF maps](https://www.kernel.org/doc/html/latest/bpf/maps.html).
```go
Probes: []*manager.Probe{
    {
        Section:          "uprobe/SSL_write",
        EbpfFuncName:     "probe_entry_SSL_write",
        AttachToFuncName: "SSL_write",
        //UprobeOffset:     0x386B0,
        BinaryPath: "/lib/x86_64-linux-gnu/libssl.so.1.1",
    },
    {
        Section:          "uretprobe/SSL_write",
        EbpfFuncName:     "probe_ret_SSL_write",
        AttachToFuncName: "SSL_write",
        //UprobeOffset:     0x386B0,
        BinaryPath: "/lib/x86_64-linux-gnu/libssl.so.1.1",
    },
    {
        Section:          "uprobe/SSL_read",
        EbpfFuncName:     "probe_entry_SSL_read",
        AttachToFuncName: "SSL_read",
        //UprobeOffset:     0x38380,
        BinaryPath: "/lib/x86_64-linux-gnu/libssl.so.1.1",
    },
    {
        Section:          "uretprobe/SSL_read",
        EbpfFuncName:     "probe_ret_SSL_read",
        AttachToFuncName: "SSL_read",
        //UprobeOffset:     0x38380,
        BinaryPath: "/lib/x86_64-linux-gnu/libssl.so.1.1",
    },
    /**/
},
```
### bash readline.so hook
hook `/bin/bash` symbol name `readline`.

# How to compile

Linux Kernel: >= 4.15.

## Tools 
* golang 1.17
* clang 9.0.0
* cmake 3.18.4
* clang backend: llvm 9.0.0   
* kernel config:CONFIG_DEBUG_INFO_BTF=y (Optional, 2022-04-17)

## command
```shell
sudo apt-get update
sudo apt-get install --yes build-essential pkgconf libelf-dev llvm-12 clang-12 linux-tools-common linux-tools-generic
for tool in "clang" "llc" "llvm-strip"
do
  sudo rm -f /usr/bin/$tool
  sudo ln -s /usr/bin/$tool-12 /usr/bin/$tool
done
git clone git@github.com:ehids/ecapture.git
cd ecapture
make
bin/ecapture --help
```

## compile without BTF
eCapture support NO BTF with command `make nocore` to compile on 2022/04/17.
```shell
make nocore
bin/ecapture --help
```


# Contributing
See [CONTRIBUTING](./CONTRIBUTING.md) for details on submitting patches and the contribution workflow.
