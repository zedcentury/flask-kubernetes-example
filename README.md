# Flask + Docker + Kubernetes + Digitalocean + nginx

1. Flaskda yozilgan proyektni dockerize qilish
2. Docker imageni digitaloceanga yuklash (Container registry)
3. Digitaloceanda Kubernetes clusterini yaratish (doctl orqali)
4. Kubernetes sozlamalarini bajarish
5. Proyektni domenga ulash
6. SSL sertifikatini ulash

## Proyektni dockerize qilish

**Dockerfile**

```dockerfile
FROM python:alpine3.19

WORKDIR /app

COPY requirements.txt .

RUN pip install -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

* Joyni tejash maqsadida proyekt uchun `python:alpine3.19` imagedan foydalanildi.
* Ish muhiti uchun `/app` tanlandi.
* Proyekt kodi ko'chirib o'tkaziladi.
* Kerakli paketlar yuklab olinadi.
* Proyekt 5000 portda ishlagani uchun shu port expose qilinadi.
* `flask run` buyrug'i bilan proyekt ishga tushiriladi.

**Imageni quramiz:**

```bash
docker build -t flask-project .
```

To'g'ri bajarilganini tekshirish uchun containerni ishga tushiramiz.

```bash
docker run -d -p 80:5000 --rm --name flask_project flask-project
```

http://localhost/ manziliga kiramiz.

```text
Hello World!
```

yozuvi ko'ringan bo'lsa, demak hammasi joyida. Containerni ishlashdan to'xtatamiz.

```bash
docker stop flask_project
```

## Docker imageni digitaloceanga yuklash (Container registry)

Imageni serverga yuklash va uni ishga tushirish uchun [Digitalocean](https://m.do.co/c/4a10c6e5754f) xizmatidan
foydalanamiz. Quyidagini bosib, digitaloceandan ro'yxatdan o'tsangiz va 60 kun uchun 200$ bonusni olishingiz mumkin.

[![DigitalOcean Referral Badge](https://web-platforms.sfo2.cdn.digitaloceanspaces.com/WWW/Badge%201.svg)](https://www.digitalocean.com/?refcode=4a10c6e5754f&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)

### Create container registry

Ro'yxatdan o'tkaningizdan keyin chap meyudan `Container Registry`ni toping va shunga kiring.

![Container Registry](https://i.imgur.com/6qUPZPW.png)

`Create a Container Registry` tugmasini bosing. Quyidagi sahifa ochiladi:

![Create a Container Registry](https://i.imgur.com/tLUtyOa.png)

Maydonga nom foydalanish uchun qulay bo'lgan nom kiriting, masalan `asliddin-container-registry`.

Regionni o'zgartirish ixtiyoriy.

Tarif sifatida boshlanishiga `Starter`ni tanlasangiz bo'ladi. Istalgan paytda tarifni o'zgartirishingiz mumkin.

Pastda tugmani bosamiz. `Container Registry` yaratildi.

### Install doctl and authenticate

`doctl` buyrug'i orqali digitaloceandagi amallarimizni terminalda boshqarishimiz mumkin. Uni o'rnatib olishdan avval
Digitaloceanda token yaratib olishimiz kerak. Buning uchun chap menyudadagi `API` menyusini topamiz va unga kiramiz.

![API](https://i.imgur.com/vpFXeVZ.png)

`Generate New Token` tugmasini bosamiz.

![Generate New Token](https://i.imgur.com/wQszuQZ.png)

`Token Name` sifatida istalgan nomni kiritishingiz mumkin. `Expiration` maydoniga token qancha muddat amal qilishi
kerakligi kiritiladi.

Oxirida esa `Write` huquqini berish orqali resurslarni yaratish, o'zgartirish va o'chirish mumkin bo'ladi.

`Generate Token` tugmasini bosamiz.

![API tokens](https://i.imgur.com/gg9l6dZ.png)

Tokenni nusxalab olib, ishonchli joyga saqlab qo'ying. Sababi token boshqa ko'rinmaydi.

Ana endi terminalda `doctl` paketini yuklab olamiz.

MacOS:

```bash
brew install doctl
```

Ana endi doctl buyrug'i orqali digitalocean akkountimizga ulanamiz. Quyidagi buyruqni ishga tushiramiz:

```bash
doctl auth init
```

Tokenni kiritib, enterni bosamiz. Digitalocean akkountimizga ulandik.

Imageni yuklashdan oldin `registry.digitalocean.com`dan ham avtorizatsiyadan o'tib olamiz.

```bash
doctl registry login
```

Ana endi imageni yuklashga o'tamiz. Avval imagening nomini o'zgartirib olishimiz kerak.
Digitaloceandagi `Container Registry` sahifasida nom tagidagi manzilni nusxalab olamiz va imagening nomini
o'zgaritiramiz:

```bash
docker tag flask-project registry.digitalocean.com/asliddin-container-registry/flask-project
```

va imageni yuklaymiz:

```bash
docker push registry.digitalocean.com/asliddin-container-registry/flask-project
```

`Container Registry` sahifasini tekshiradigan bo'lsak, image yuklanganini ko'rishimiz mumkin.

![Images](https://i.imgur.com/RCWHJcD.png)

### Create Kubernetes cluster using doctl

```bash
doctl kubernetes cluster create flask-cluster
```

Digitaloceanda `flask-cluster` nomli cluster yaratiladi. Ushbu buyruq bir necha daqiqa vaqt oladi.

```bash
kubectl config current-contexts
```

Agar `do-nyc1-flask-cluster` bo'lsa, demak hammasi aks holda quyidagi komandani amalga oshiring.

```bash
kubect config use-context do-nyc1-flask-cluster
```

### Configure Kubernetes resource files

`kubernetes/deployment.yaml` faylida image manzilini o'zgartishingiz kerak.

Undan keyin `Container Registry` sahifasiga kirib, cluster bilan integratsiya qilishimiz kerak.

![Integrating with Kubernetes](https://i.imgur.com/4FosfUV.png)

`Integrating with Kubernetes`ga o'tamiz va `flask-cluster` belgilab, `Save & Continue` tugmasini bosamiz.

Keyin quyidagi komandani ishga tushiramiz:

```bash
kubectl apply -f kubernetes/deployment.yaml -f kubernetes/service.yaml
```

Servislar ro'yxatini tekshiramiz:

```bash
kubectl get services
```

Natija

```text
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
flask-service   LoadBalancer   10.245.201.66   <pending>     80:30979/TCP   15s
kubernetes      ClusterIP      10.245.0.1      <none>        443/TCP        8m46s
```

`flask-service` uchun `EXTERNAL-IP` `<pending>` turganini ko'rishingiz mumkin. Digitalocean sizga ma'lum vaqtdan IP
manzil ajratadi. Bu bir necha daqiqadan bir necha soatgacha vaqt olishi mumkin.

IP manzil ajratilgandan keyin servislar ro'yxatini tekshirib ko'rsangiz quyidagiga o'zgargan bo'ladi:

```text
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
flask-service   LoadBalancer   10.245.84.70   146.190.196.78   80:31891/TCP   2m39s
kubernetes      ClusterIP      10.245.0.1     <none>           443/TCP        30m
```

Sizdagi IP manzil boshqacha bo'ladi. Istalgan browser orqali shu manzilga kirsangiz,

```text
Hello World!
```

matni ko'rinadi.

### Proyektni domenga ulash

Terminalda `ingress-nginx`ni sozlab olinadi:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.6/deploy/static/provider/do/deploy.yaml
```

Quyidagi komanda orqali `ingress-nginx` joriy IP manzilimizga bog'langanini tekshiramiz:

```bash
kubectl get svc --namespace=ingress-nginx
```

Natijani tekshirsak, `EXTERNAL-IP` ustunidagi qiymat `<pending>` bo'lishi mumkin. Bu biroz vaqt oladi.

Ungacha `kubernetes/ingress.yaml` faylini sozlab chiqamiz.

Bu faylda siz faqat `host` qiymatini, ya'ni domen nomini o'zingiznikiga o'zgartirishingiz kerak.

Buning uchun siz avval internetdan domen nomini sotib olishingiz va name serverlarni quyidagiga o'zgartirishingiz kerak:

```text
dns2.p06.nsone.net
dns4.p06.nsone.net
dns1.p06.nsone.net
dns3.p06.nsone.net
```

Shuningdek digitaloceanda ham domenni ro'yxatdan o'tkazishingiz kerak. Buning uchun digitaloceanda chap
menyuda `Networking` sahifasi o'tishingiz kerak.

![Networking domains](https://i.imgur.com/Uyn3vKC.png)

`Domains` bo'limiga o'tamiz, o'z domenimizni kiritamiz (masalan, `apple.uz`) va `Add Domain` tugmasini bosamiz.

```bash
kubectl get svc --namespace=ingress-nginx
```

orqali `EXTERNAL-IP` qiymatini tekshiramiz. Agar IP manzil berilgan bo'lsa, ishni davom ettiramiz.

`kubernetes/ingress.yaml` faylida `host` qiymatini o'z domenimizga o'zgartirgach, quyidagi komandani ishga tushiramiz:

```bash
kubectl apply -f kubernetes/ingress.yaml
```

Biroz vaqtdan keyin browserda domen manziliga kiradigan bo'lsangiz,

```text
Hello World!
```

matni ko'rinadi.

### SSL sertifikatini ulash

`cert-manager`ni sozlab olamiz:

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.14.2/cert-manager.yaml
```



# DigitalOcean token

dop_v1_09da56fbfc8b1d416b955432db77ee7a600b88ba8305e04628323cce128b1d14
