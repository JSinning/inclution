Deplegamos maquina de dockerlabs con

`sudo ./auto_deploy.sh inclusion.tar`

hacemos un brarido con **nmap** para lista los puertos abiertios usando 

`sudo nmap -p- --open -sS -sCV --min-rate 5000 -n -Pn -vvv 172.17.0.2 -oN scaning.txt`

![Pasted image 20240729151923](https://github.com/user-attachments/assets/1e4e6bcb-55ed-4986-a150-3fbf0cce05bc)

Ok lugo de hacer el barido de puertos obsevamos que la maquina tiene abiertos el puerto **22, 80** puertos comunes que ya conecemos. Analisando cada puerto obesrvamos que puerto 22 **SSH** con una version de **OpenSSH  9.2p1** no tan actual pero de igunamente no nos permite usar exploits para enuemar usaurios como lo es el  **Username Enumeration**. y finalmente  el puerto 80 **http** en el cual vemos en que tien un titulo comun de la paguina de muestra de apache  en su sitio web ya que es un **APACHE  2.4.57**  asi que vamos al navegador y veamos.

![Pasted image 20240729152310](https://github.com/user-attachments/assets/e6dbf563-9a6b-4c8e-9d3f-20426459c976)

nada estraño ni haciendo CTRL + U vemos comentarios que nos de pistas asi que recurimos a gobuster para realizar fuzzing y que otras rutas hay en el sito web. 

`gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r`

![Pasted image 20240729152738](https://github.com/user-attachments/assets/7196372d-1438-45fe-858c-4f3ef2a4bc7e)

vemos que gobuster  nos encontro dos rutas una con estado **200 OK**  llamada  **shop** y un **server-status** no tan interesante asi que nos dirigimos a la ruta *shop*

![Pasted image 20240729153133](https://github.com/user-attachments/assets/cb793863-9b4c-41b5-a8cc-e0c6038bbe1b)
 analizando la paguina vemos que es una imgen de un teclado y un titilo que dice *Tienda de Teclado*  pero lo vemos que es intereando es lo que hay que la esquina inferior izquierda de la pantalla vemo un erro que dice Error de Sistema y vemos que trata de llamar a un archivo del sistema por el metodo GET esto meda un idea de implementar algo como **URL/?archivo=index.php** porque php porque el Wappalyzer no indica **PHP**.

![Pasted image 20240729154229](https://github.com/user-attachments/assets/94f1061a-0f03-4bd2-a887-ed6ceaaad588)

OK,  como sabemos esto de archivos es un copsecto de path Travesal con LFI, asi que listaremos el el /etc/passwd con el siguiente link

`http://172.17.0.2/shop/?archivo=../../../../etc/passwd` 
 y obtenemos el siguiente resultado
![Pasted image 20240729163032](https://github.com/user-attachments/assets/466d8ef3-b84e-4c63-85a9-a83bc89b74ca)

si vemos esto es la info de passwd y vemos usuarios con **root** **www-data**,  **seller**, **manchi** uno muy interesante sabiendo que estan estos usuarios intentaremos hacer un brute force para inicar secion en SSH con Hydar

`hydra -l seller -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4`
`hydra -l manchi -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4`

![Pasted image 20240729163728](https://github.com/user-attachments/assets/01921485-2620-4af8-be95-cf0eedb66820)

bingo ya que con el usuario **seller** no tuvimos suerte intentamos con el usuario **manchi** y obtivimos un un password que podemos usar el cual es **lovely**  

asi que  iniciamos secion en ssh con 

`shh manchi@172.17.0.2` 

![Pasted image 20240729164142](https://github.com/user-attachments/assets/e8d0c47a-21a4-4d81-90a0-0c49965d1cc6)

y ya somos el usurio manchi ahor vemos si tenmos algun arcivo sudo con **sudo -l** pero no tenemos suerte asi que buscamos alguno alchivo **SUID** con `find / -perm -4000 -ls 2>/dev/null`  pero no tenemos nada.  asi que en esta aocacion usaremos el archivo  de fuerza bruta  [Maalfer/Sudo_BruteForce: Script hecho en bash para realizar un ataque de fuerza bruta a un usuario de un sistema Linux. (github.com)](https://github.com/Maalfer/Sudo_BruteForce/tree/main)   para realizar un ataque a un usuario.  los paso que arremos son los siguiente.


con este primer comando desacrmos el script a nuestra maquina actacante. 

`wget https://raw.githubusercontent.com/Maalfer/Sudo_BruteForce/main/Linux-Su-Force.sh` 

y el segundo copiamos el rockyou. 

`cp /usr/share/wordlists/rockyou.txt .` 

en vista que la maquina no tiene ni curl y wget para transferir archivo usaremos scp 

`scp rockyou.txt manchi@172.17.0.2:/home/manchi/rockyou.txt
`scp Linux-Su-Force.sh manchi@172.17.0.2:/home/manchi/rockyou.txt``

con el primero copiamos el rockyou y el segundo el bash para crackear el password de usurios seller. en abas acaciones escribiel password de manchi.

![Pasted image 20240729170927](https://github.com/user-attachments/assets/4d33c6a3-ebc1-4fc8-be5d-ddbb18d02313)

![Pasted image 20240729171031](https://github.com/user-attachments/assets/b89f4f16-d1cb-4c4a-bd40-a9a57e9476c6)

ahora como podemo ver en la maquina victima el ya estan los archivos asi que ahora ejecutamos el archivo de la siguinte manera

`./Linux-Su-Force.sh seller rockyou.txt` 

![Pasted image 20240729171535](https://github.com/user-attachments/assets/71b78946-a584-4cd4-bb0f-a8fff55ca405)

y vemos que obtenemos un password **qwerty** . algunas recomendaciones si no encuntra el password a la primera ejecuten varias veces el script.

ahora con inicamos secion con seller 

`su seller` 

![Pasted image 20240729171757](https://github.com/user-attachments/assets/7b6cb3d6-87e2-4f15-ae56-e9f48d370f54)

y ahora de nuevo usamso **sudo -l** para ver que permiso sudo tenemos para ejecutar encontramos  **/usr/bin/php**  asi que vamos GTFOBins  y buscamos php

![Pasted image 20240729172101](https://github.com/user-attachments/assets/9a953614-b6ca-4741-b99f-2808f9e461c8)
y vemos que comi sudo podemo escalar privilejios usando  

`sudo -u root /usr/bin/php -r "system('/bin/bash');"`

![Pasted image 20240729172642](https://github.com/user-attachments/assets/f6f2eae1-7105-40af-ac6e-22714e8efeb8)


y asi consegimo el root en el machina inclution

`by juandas13` 














 
