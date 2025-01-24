# RPM Tools

## Query RPM Package

``` bash
$ rpm -qp --scripts EcoX-5.13.0-9cdad8a.el9.x86_64.rpm
preinstall program: /bin/sh
postinstall scriptlet (using /bin/sh):
/sbin/ldconfig
postuninstall scriptlet (using /bin/sh):
rm -rf /opt/ecox-5.13.0
rm -rf /etc/ld.so.conf.d/ecox.conf
rm -rf /etc/profile.d/ecox.sh
/sbin/ldconfig
$ rpm -qp --list EcoX-5.13.0-9cdad8a.el9.x86_64.rpm
/opt/ecox-5.13.0
/opt/ecox-5.13.0/bin
/opt/ecox-5.13.0/bin/EcoX
/opt/ecox-5.13.0/etc
/opt/ecox-5.13.0/etc/ecox.conf.sample
/opt/ecox-5.13.0/etc/log4crc.sample
```

## Extract Files From RPM Package

```
$ rpm2cpio EcoX-5.13.0-9cdad8a.el9.x86_64.rpm | cpio -idmv
```
