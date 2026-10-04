# Segundo Parcial: Servicios Telemáticos

Universidad Autónoma de Occidente, Facultad de Ingeniería. Código: 2230183. Sustentación: 6 de octubre de 2026.

Configuraciones usadas en el parcial: FTPS protegido por firewall UFW, DNS sobre TLS (DoT) y SFTP protegido por UFW.

## Topología

| Rol en el parcial | VM de Vagrant | Red pública (cliente ↔ servidor 1) | Red privada (servidor 1 ↔ servidor 2) |
|---|---|---|---|
| Servidor 1 (UFW, único punto de entrada) | `servidor` | 192.168.56.3 (eth1) | 192.168.50.3 (eth2) |
| Servidor 2 (vsftpd y OpenSSH) | `cliente` | no tiene | 192.168.50.2 (eth1) |
| Cliente Linux (DoT, sftp, openssl) | `esclavo` | 192.168.56.4 (eth1) | no tiene |

El cliente Windows (WinSCP y Wireshark) usa la red 192.168.56.0/24. La red 192.168.50.0/24 es una red interna de VirtualBox (`red-privada`), por lo que el Servidor 2 no es alcanzable directamente desde los clientes: solo se llega a él a través del reenvío de puertos del Servidor 1.

## Contenido del repositorio

- `servidor1/before.rules`: reglas NAT (DNAT y MASQUERADE) de UFW.
- `servidor1/ufw_status.txt`: salida de `ufw status verbose`.
- `servidor1/iptables_nat.txt`: salida de `iptables -t nat -L -n -v`.
- `servidor2/vsftpd.conf`: configuración de FTPS con TLS explícito.
- `servidor2/sshd_config`: configuración de OpenSSH con usuario SFTP enjaulado.
- `servidor2/ca.crt`: certificado público de la CA propia generada con OpenSSL.
- `esclavo/resolved.conf`: configuración de DNS sobre TLS con systemd-resolved.

## Primera parte: FTPS protegido por UFW

**Servidor 1.** Política por defecto `deny` para tráfico entrante y enrutado, reenvío IP habilitado y permitido solo lo necesario:

- `22/tcp` hacia el propio servidor (administración SSH).
- Reglas `ufw route` desde eth1 hacia eth2 a 192.168.50.2: puerto `21/tcp` (control FTP), rango `50000:50010/tcp` (datos pasivos) y `22/tcp` (SFTP, tercera parte).
- En `before.rules`, el bloque `*nat` redirige con DNAT el puerto 21, el rango 50000:50010 y el puerto 2222 hacia 192.168.50.2 (el 2222 al puerto 22), y aplica MASQUERADE hacia el Servidor 2 por eth2.

**Servidor 2.** vsftpd en modo FTPS explícito: `ssl_enable=YES`, `force_local_logins_ssl=YES`, `force_local_data_ssl=YES`, SSLv2, SSLv3 y TLS 1.0 deshabilitados, `require_ssl_reuse=NO` para compatibilidad con WinSCP, y modo pasivo con `pasv_min_port=50000`, `pasv_max_port=50010` y `pasv_address=192.168.56.3`.

**Certificados.** CA propia y certificado de servidor con la IP 192.168.56.3 en el SAN, ambos generados con OpenSSL. Verificación con `openssl s_client -starttls ftp`: TLS 1.3, suite `TLS_AES_256_GCM_SHA384` y `Verify return code: 0 (ok)`.

## Segunda parte: DNS sobre TLS

En `esclavo/resolved.conf`: `DNS=1.1.1.1#cloudflare-dns.com 8.8.8.8#dns.google`, `FallbackDNS=1.0.0.1#cloudflare-dns.com 8.8.4.4#dns.google` y `DNSOverTLS=yes`.

- `resolvectl status` muestra `+DNSOverTLS` en la sección `Global`.
- Con DoT activo, la captura con `tcp.port == 853` muestra el handshake TLS 1.3 y datos cifrados hacia 1.1.1.1 y 8.8.8.8, y `udp.port == 53` queda vacío.
- Con `DNSOverTLS=no`, la consulta viaja en claro por UDP/53 y se leen el dominio y las IPs de la respuesta.
- El DHCP de VirtualBox añade DNS en eth0; se usa `resolvectl default-route eth0 no` para que las consultas usen solo los resolvers globales.

## Tercera parte: SFTP protegido por UFW

En `servidor2/sshd_config`, el usuario `sftp_2230183` tiene un bloque `Match User` con `ChrootDirectory /srv/sftp/sftp_2230183`, `ForceCommand internal-sftp`, `AllowTcpForwarding no` y `X11Forwarding no`. La raíz de la jaula pertenece a `root` y el usuario escribe en la subcarpeta `subidas`.

- El Servidor 1 publica el puerto 2222 y lo reenvía al 22 del Servidor 2.
- Sin la regla `ufw route` la conexión da timeout; con la regla, `sftp -P 2222 sftp_2230183@192.168.56.3` funciona.
- Un `ssh` con ese usuario es rechazado con `This service allows sftp connections only.`
- La captura con `tcp.port == 2222` muestra una sola conexión TCP: intercambio de versiones SSH-2.0, intercambio de claves y el resto cifrado.

## Notas

- No se incluyen las claves privadas (`ca.key`, `servidor.key`) ni las capturas `.pcap`.
- Los archivos son copias de los que están en `/etc` de cada máquina.
