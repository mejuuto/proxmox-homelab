# Configuraciones de referencia

Archivos y comandos usados en el hardening de Proxmox VE. Adapta las IP a tu red y **prueba cada cambio con una segunda sesión abierta** para no quedarte fuera.

## SSH: solo clave y límites de sesión

`/etc/ssh/sshd_config.d/99-hardening.conf`:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
MaxAuthTries 3
MaxSessions 2
LogLevel VERBOSE
ClientAliveCountMax 2
AllowTcpForwarding no
X11Forwarding no
AllowAgentForwarding no
TCPKeepAlive no
```

Comprobar y aplicar:

```
sshd -t && systemctl reload ssh
```

## fail2ban

`/etc/fail2ban/jail.local`:

```
[DEFAULT]
backend = systemd
ignoreip = 127.0.0.1/8 192.168.0.21

[sshd]
enabled = true
maxretry = 3
bantime = 1h
```

Estado:

```
fail2ban-client status sshd
```

## Parches automáticos y verificación de paquetes

```
apt install -y fail2ban needrestart unattended-upgrades debsums apt-show-versions libpam-tmpdir
```

`/etc/apt/apt.conf.d/20auto-upgrades`:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

## Protocolos de red no usados

`/etc/modprobe.d/99-hardening-protocols.conf`:

```
install dccp /bin/false
install sctp /bin/false
install rds /bin/false
install tipc /bin/false
```

## Parámetros del kernel

`/etc/sysctl.d/99-hardening.conf`:

```
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.default.accept_redirects=0
net.ipv6.conf.all.accept_redirects=0
net.ipv6.conf.default.accept_redirects=0
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.all.log_martians=1
net.ipv4.conf.default.log_martians=1
net.ipv4.conf.all.rp_filter=1
fs.protected_fifos=2
kernel.kptr_restrict=2
kernel.sysrq=0
kernel.unprivileged_bpf_disabled=1
net.core.bpf_jit_harden=2
```

Aplicar con `sysctl --system`. No se usa `kernel.modules_disabled=1` porque rompería Proxmox.

**Nota:** tras aplicarlo, `kernel.sysrq` y `kernel.kptr_restrict` toman el valor configurado, pero `net.ipv4.conf.all.rp_filter` sigue en `2` (modo laxo). Se deja así y se documenta como desviación respecto a la sugerencia de Lynis.

## Banner de Postfix

```
postconf -e 'smtpd_banner = $myhostname ESMTP'
systemctl restart postfix
```

## Firewall de Proxmox

Reglas recomendadas a nivel de Datacenter:

1. Alias `admin-pc` con la IP del equipo de administración y `lan` con la red local.
2. Permitir TCP 8006 y SSH solo desde `admin-pc`; permitir ping desde `lan`.
3. Política de entrada en `DROP`, activándola **después** de crear las reglas de acceso.

Prueba de funcionamiento:

- Desde el equipo de administración: `Test-NetConnection 192.168.0.50 -Port 8006` debe conectar.
- Desde otra máquina de la red: `curl -k --max-time 5 https://192.168.0.50:8006` debe dar timeout.
