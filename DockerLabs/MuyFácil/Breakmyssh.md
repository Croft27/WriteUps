# Reconocimiento

Una vez desplegada la máquina comprobamos que tenemos conectividad con ``ping -c1 172.17.0.2``

![image](https://github.com/user-attachments/assets/66afed69-1a9a-46ae-b398-4836e6d49a25)

``-c1``> solo repite el ping 1 sola vez.

Una vez tenemos conectividad con la maquina Víctima, realizo un reconomiento con ``Nmap``

```bash
nmap -p- --open -sS -sV -sC -min-rate 5000 -n -Pn 172.17.0.2 -oN scan
```

<details>
	<summary><strong>Explicación del comando:</strong></summary>

### 1️⃣ `-p-` → Escaneo de **todos los puertos (1-65535)**
- Sin este flag, Nmap solo escanea los **1000 puertos más comunes**.
- Con `-p-`, Nmap escanea **los 65535 puertos**, lo que permite detectar más servicios.

---

### **2️⃣ `--open` → Solo muestra puertos abiertos**
- Filtra los resultados para que solo se muestren **puertos abiertos**, ocultando los que están cerrados o filtrados.

---

### 3️⃣ `-sS` → Escaneo **TCP SYN (Stealth Scan)**
- Es un escaneo **sigiloso** que **no establece conexiones completas** (envía solo paquetes SYN y analiza las respuestas).
- Ideal para **evadir detecciones** en firewalls o sistemas de detección de intrusos (**IDS**).

---

### **4️⃣ `-sV` → Detección de versiones de los servicios**
- Intenta identificar qué **servicio exacto** está corriendo en cada puerto abierto (ejemplo: `Apache 2.4.41` o `OpenSSH 8.2`).

---

### **5️⃣ `-sC` → Ejecución de scripts básicos**
- Activa los **Nmap Scripting Engine (NSE)** con los scripts predeterminados (`default`).
- Algunos scripts útiles que se ejecutan:
  - Detección de versiones (`version`).
  - Consulta de banners (`banner`).
  - Pruebas de seguridad básicas (`default`).

---

### **6️⃣ `--min-rate 5000` → Fuerza un escaneo más rápido**
- **Establece un mínimo de 5000 paquetes por segundo**, lo que hace que el escaneo sea más **rápido y agresivo**.
- Puede generar **detección en firewalls o IDS**, ya que envía muchas solicitudes en poco tiempo.

---

### **7️⃣ `-n` → No usa resolución de DNS**
- Evita que Nmap intente resolver nombres de dominio **para acelerar el escaneo**.

---

### **8️⃣ `-Pn` → Omite la detección de hosts**
- Nmap **no realiza un `ping` previo** para ver si la máquina está activa, simplemente **asume que está encendida** y procede con el escaneo.
- Útil si el host bloquea **ICMP (`ping`)** pero tiene puertos abiertos.

---

### **9️⃣ `172.17.0.2` → IP objetivo**
- Es la dirección de la máquina que se está escaneando.

---

### **🔟 `-oN Scan` → Guarda los resultados en un archivo**
- `-oN` guarda la salida en un archivo en formato **legible (`Scan`)** para su posterior análisis.

---
</details>

![image](https://github.com/user-attachments/assets/79e7d500-7636-4d7a-beae-c67f3f0c3188)

Para ver los resultados de una forma mas clara realizo el siguiente comando: ``cat Scan -l python``

![image](https://github.com/user-attachments/assets/4293f437-f3a0-4b2f-8ebe-f46f95ba29fc)

Como se muestra en el resultado obtenemos El puerto 22 como puerto abierto y con sus versiones.

Como solo tenemos como resultado el puerto 22 abierto, ejecuto `searchsploit OpenSSH 7.7` para ver si existen vulnerabilidades en esta versión.

![image](https://github.com/user-attachments/assets/51d251d0-6ce3-48a4-b357-752d866e1fcb)

Como se puede observar en la imagen existen 3 exploit para dicha versión de OpenSSH

En mi caso voy a ejecutar la insterfaz de Metasploit `msfconsole` donde buscaremos módulos relacionados con OpenSSH.

![image](https://github.com/user-attachments/assets/26c061be-14e0-4c0a-8aea-60ae7d5b16ad)

Como se muestra en la imagen tenemo diferentes módulos. Vamos a elegir la opcion 3 que nos permite cargar el módulo para enumerar usuarios por ssh.

![image](https://github.com/user-attachments/assets/efaabfcf-cd7e-434c-909b-d729773f5bf8)

Elegido el módulo veremos sus opciones con `show options` y cargaremos la información necesaria para poder ejecutar el Exploit.

![image](https://github.com/user-attachments/assets/a3082feb-fdbc-4a14-a4ae-1bebbbf43734)

En este caso solo tendremos que indicar el RHOST con la ip de la máquina victima ``172.17.0.2`` y el USER_DILE con el diccionarió que queremos utilizar ``/usr/share/wirdlist/metasploit/unix_users.txt``

![image](https://github.com/user-attachments/assets/a9c64a76-2b55-4c4b-90d0-36643071fd99)

Añadida ya toda la información necesaria dentro del modulo, ejecutamos el exploit con ``run``

![image](https://github.com/user-attachments/assets/036c4d82-00b3-43e8-a581-bfcbab72917d)

Obtenemos el siguiente resultado:

![image](https://github.com/user-attachments/assets/b3757544-bd81-4d7f-9570-290caff740dd)

Tenemos diferentes usuario enumerados donde el que mas llama la atención es el usuario ``ROOT``.
Obtenido dicho usuario, realizamos un ataque de fuerza bruta con ``Hydra``

## **Ataque de fuerza bruta a SSH con Hydra**

Probamos credenciales en el puerto **22 (SSH)** con **Hydra**:

Usamos `Hydra` con el usuario root:

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://172.18.0.2
```
![image](https://github.com/user-attachments/assets/44fef571-8286-4bed-b077-dfd366b7f7b6)

Obtenemos la contraseña **`estrella`** para el usuario `root`.

## **Conexión SSH**

```bash
ssh root@172.17.0.2
```
![image](https://github.com/user-attachments/assets/c86b30a5-ab29-433d-87ec-217bf984a048)

Si realizamos el comando ``whoami`` vemos que no hace falta realizar una escalada de privilegios ya que hemos iniciado la conexión como Root.

![image](https://github.com/user-attachments/assets/24996776-7930-41e7-9078-61e740b22283)

:whale: :white_check_mark:


