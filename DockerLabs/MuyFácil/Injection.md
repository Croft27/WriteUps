# Reconocimiento

Una vez desplegada la máquina comprobamos que tenemos conectividad con ``ping -c1 172.17.0.2``

![image](https://github.com/user-attachments/assets/28ace12f-fb05-4a64-9861-e64a60a83609)

``-c1``> solo repite el ping 1 sola vez.

Una vez tenemos conectividad con la maquina Víctima, realizo un reconomiento con ``Nmap``

```bash
nmap -p- --open -sS -sV -sC -min-rate 5000 -n -Pn 172.17.0.2 -oN Puertos
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

### **🔟 `-oN Puertos` → Guarda los resultados en un archivo**
- `-oN` guarda la salida en un archivo en formato **legible (`Puertos`)** para su posterior análisis.

---
</details>

![image](https://github.com/user-attachments/assets/a263daea-40f8-473a-b321-673d5950cfe1)

Para ver los resultados de una forma mas clara realizo el siguiente comando: ``cat Puertos -l ruby``

![image](https://github.com/user-attachments/assets/d33f6daa-a253-400f-b5fc-6184c4b84204)

Como se muestra en el resultado obtenemos los puertos 22 y 80 como puertos abiertos y con sus versiones.

# Puerto 80 (HTTP)

Accedemos al servidor web ingresando la dirección IP en el navegador:

![image](https://github.com/user-attachments/assets/6b37498f-4d5c-4220-bbf2-14dcbc0b7e3e)

Obtenemos una pagina con un Login.

Procedo a realizar una Injeccion Sql añadiendo en el USER: ``admin' or 1=1; --``

![image](https://github.com/user-attachments/assets/ed7d6b55-021a-4c4e-a1c3-c5718cf1e2a1)

<details>
	<summary><strong>Explicación de SQL Injection:</strong></summary>

### **📌 Explicación paso a paso:**

1️⃣ **`admin'`** → Cierra prematuramente la cadena de texto esperada para el nombre de usuario.

2️⃣ **`or 1=1`** → Introduce una condición **siempre verdadera**, lo que hace que la autenticación sea exitosa sin importar la contraseña.

3️⃣ **`; --`** →
   - **`;`** Finaliza la consulta actual.
   - **`--`** Comenta el resto de la consulta SQL, ignorando cualquier validación de contraseña.

### **🔥 Ejemplo en una consulta SQL típica**

Una aplicación podría ejecutar una consulta similar a esta:

```sql
SELECT * FROM usuarios WHERE username = 'admin' AND password = 'password';
```

Si el atacante introduce:

```sql
admin' or 1=1; --
```

La consulta final se transforma en:

```sql
SELECT * FROM usuarios WHERE username = 'admin' OR 1=1; -- ' AND password = 'password';
```

Dado que **`1=1` siempre es verdadero**, la consulta devuelve todos los registros, permitiendo al atacante acceder **sin necesidad de conocer la contraseña**.
</details>

Tenemos un resultado positivo el cual nos devuelve lo siguiente:

![image](https://github.com/user-attachments/assets/8d31134b-d4f1-4d06-bec6-a4b09dd33a93)

Con este usurio y contraseña, realizamos una conexión SSH.

# Puerto 22 (SSH)

```bash
ssh dylan@172.17.0.3
```

![image](https://github.com/user-attachments/assets/90b5e236-9eef-449f-a4bc-1dc081305ea4)

# Escalada de privilegios.

Ejecutamos: ``sudo -l``

![image](https://github.com/user-attachments/assets/5a168e24-5763-48bf-98a1-35e4a56e7886)

El usuario Dylan no tiene ningun permiso para usar sudo.
En vista a que no tiene permisos, buscamos comandos con ``SUID`` habiliutado usando el siguiente comando:

```bash
find / -perm -4000 2>/dev/null
```
<details>
	<summary><strong>Explicación del comando: find / -perm -4000 2>/dev/null</strong></summary>

El comando `find / -perm -4000 2>/dev/null` se utiliza para **buscar archivos con el bit SUID activado** en todo el sistema de archivos.

### **📌 Explicación paso a paso:**

1️⃣ **`find /`** → Inicia la búsqueda desde la raíz (`/`), es decir, en todo el sistema de archivos.

2️⃣ **`-perm -4000`** → Busca archivos con el **bit SUID** activado (`4000`).
   - El bit **SUID (Set User ID)** permite que un archivo se ejecute con los privilegios de su propietario (generalmente `root`).
   - Esto significa que si un usuario ejecuta un archivo con SUID, este se ejecutará con los permisos del propietario en lugar de los del usuario.

3️⃣ **`2>/dev/null`** → Redirige los errores (`stderr`, descriptor `2`) a `/dev/null` para evitar mensajes de acceso denegado en directorios protegidos.
</details>

![image](https://github.com/user-attachments/assets/1cdd9053-3adb-46f7-a7ee-7996202c8feb)

Vemos que tenemos el binario /env con el bit SUID activado.
Este binario imprime o modifica variables de entorno.

Buscamos en la página web CTFOBins la posible escalada de privilegios con este binario:

![image](https://github.com/user-attachments/assets/d41ae851-e472-4dd7-9b16-940346aa2da8)

En la terminal, nos dirigimos al directorio ``/usr/bin``.
Dentro de el ejecutamos el siguiente comando que se nos muestra en la anterior imagen:

```bash
./env /ben/sh -p
```
![image](https://github.com/user-attachments/assets/b261fd23-9a1c-45f5-b765-ccf427b00e01)

Realizamos un ``whoami`` y vemos como hemos conseguido escalar privilegios y ser ``ROOT``

![image](https://github.com/user-attachments/assets/5d5a80e4-f1ee-4d06-be3c-01b491c95119)









