# Laboratorio Zero Trust de la sesión 13

Práctica de Arquitectura de Sistemas Computacionales de la UNIAJC. Evaluamos la identidad de los buses de MetroMIO, la protección de datos de HospitValle y la conectividad de AgroSugar. Todas las pruebas con PostgreSQL usan datos sintéticos.

| Integrante | Responsabilidad |
|---|---|
| Josen Julian Mina Carabali | Líder técnico y gestor del repositorio |
| Fabio Tenorio Angulo | Auditor |
| Josue Mindineros Castillo | Documentador |

## Estructura del repositorio

```
zerotrust-sesion13/
├── README.md
├── .gitignore
├── kind-zerotrust.yaml          # Clúster kind de 2 nodos sin CNI por defecto
├── reto1-mtls/
│   ├── pki/                     # Extensiones X.509 y certificados (.crt). Las llaves NO se versionan
│   ├── nginx/nginx.conf         # Nginx con mTLS y TLS 1.3
│   └── scripts/emitir-bus.sh    # Emisión de certificados de buses en lote
├── reto2-hospitvalle/
│   ├── luks/                    # (los .img, .keyfile y .bak no se versionan)
│   └── k8s/
│       ├── 00-namespace.yaml        # Namespace con Pod Security Admission "restricted"
│       ├── 01-sealed-pg-uci-cred.yaml  # Credencial sellada (texto cifrado)
│       ├── 02-rbac.yaml             # ServiceAccount, ConfigMap, Role y RoleBinding
│       └── 03-pg-uci.yaml           # Deployment endurecido y Service
├── reto3-agrosugar/
│   ├── k8s/
│   │   ├── 01-escenario.yaml        # Broker MQTT, base de datos, sensor e intruso
│   │   └── 02-networkpolicies.yaml  # default-deny + 3 políticas de permitidos
│   └── scripts/matriz.sh        # Matriz de conectividad
├── evidencias/                  # Capturas de los retos y video de sustentacion
└── informe/                     # Informe PDF y anexos
```

## Requisitos y preparación

Para repetir el laboratorio necesitas Bash en Linux, Git, Docker, OpenSSL 3, curl, kind y kubectl. Para la prueba de disco usamos una VM Linux con cryptsetup, dispositivos loop, ext4 y permisos de administración. Para la prueba de secretos necesitas kubeseal y su controlador. Trabajamos con OpenSSL 3.5.5, kubectl v1.37.2 y kind v0.30.0.

Abre una terminal en la raíz del repositorio. Para crear el clúster de prueba, ejecuta estos comandos.

```bash
kind create cluster --config kind-zerotrust.yaml
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/calico.yaml
kubectl wait --for=condition=Ready nodes --all --timeout=300s
kubectl get nodes -o wide
kubectl get pods -n kube-system -l k8s-app=calico-node
```

El archivo de kind desactiva el CNI predeterminado para que podamos usar Calico. Antes de seguir, comprueba que los dos nodos estén listos y que haya un pod de Calico en ejecución en cada uno.

## 1. MetroMIO

Sigue los pasos 2.1 a 2.4 de la guía para crear la CA y emitir los certificados del servidor y del bus. Usa ingress.metromio.net como SAN del servidor y clientAuth para el bus. Si empiezas desde un clon nuevo, genera las llaves y sus certificados en la misma copia local. Guarda las llaves privadas fuera de Git. Cuando tengas la PKI, verifica los certificados, inicia Nginx y prueba la conexión.

```bash
(cd reto1-mtls/pki && openssl verify -CAfile ca.crt ingress.crt bus-101.crt)
docker run -d --name ingress-mtls -p 8443:8443 \
  -v "$PWD/reto1-mtls/nginx/nginx.conf:/etc/nginx/nginx.conf:ro" \
  -v "$PWD/reto1-mtls/pki:/etc/nginx/pki:ro" nginx:1.27-alpine
p=reto1-mtls/pki
curl --cacert "$p/ca.crt" --cert "$p/bus-101.crt" --key "$p/bus-101.key" \
  --resolve ingress.metromio.net:8443:127.0.0.1 -w '\nHTTP %{http_code}\n' \
  https://ingress.metromio.net:8443/telemetria
```

En el laboratorio, Nginx respondió HTTP 200 y SUCCESS a bus-101. Sin certificado respondió HTTP 400 y con un certificado de otra CA también devolvió 400. Cuando limitamos el cliente a TLS 1.2, curl terminó con el código 35. Para emitir los certificados de la flota usamos `reto1-mtls/scripts/emitir-bus.sh`.

## 2. HospitValle

**Disco en la VM.** Sigue los pasos 4.1 a 4.7 con dos imágenes nuevas de 512 MiB. Prepara una con LUKS2 y ext4 y la otra solo con ext4. Monta cada imagen por separado, inserta el marcador sintético en PostgreSQL 16 y ejecuta CHECKPOINT. Antes de comparar, detén PostgreSQL, desmonta las imágenes y cierra el volumen LUKS. Ejecuta la búsqueda desde la carpeta donde guardaste los archivos.

```bash
strings uci-cifrado.img | grep -c 'PACIENTE-SINTETICO'
strings uci-plano.img | grep -c 'PACIENTE-SINTETICO'
```

Obtuvimos 0 líneas con el marcador en el disco cifrado y 2 en el disco plano. En otra ejecución puede variar el segundo conteo. Comprueba que el marcador aparezca en la imagen plana y que la imagen LUKS esté cerrada cuando hagas la búsqueda.

**Pod en Kubernetes.** Aplica el namespace e instala el controlador de Sealed Secrets como indica el paso 5.3. Si creaste otro clúster, sella la credencial sintética con la llave pública de su controlador. Usa el nombre pg-uci-cred y el namespace hospitvalle. Comprueba que el controlador haya creado el Secret antes de desplegar PostgreSQL.

```bash
kubectl apply -f reto2-hospitvalle/k8s/00-namespace.yaml
# Una vez instalado el controlador y preparado el manifiesto para este clúster:
kubectl apply -f reto2-hospitvalle/k8s/01-sealed-pg-uci-cred.yaml
kubectl -n hospitvalle get secret pg-uci-cred
kubectl apply -f reto2-hospitvalle/k8s/02-rbac.yaml
kubectl apply -f reto2-hospitvalle/k8s/03-pg-uci.yaml
kubectl -n hospitvalle rollout status deploy/pg-uci
kubectl -n hospitvalle exec deploy/pg-uci -- id
SA=system:serviceaccount:hospitvalle:uci-db-sa
kubectl auth can-i get configmap/uci-config -n hospitvalle --as="$SA"
kubectl auth can-i get secrets -n hospitvalle --as="$SA"
kubectl auth can-i delete pods -n hospitvalle --as="$SA"
```

Dentro del pod obtuvimos UID 10001. La cuenta de servicio pudo leer el ConfigMap, pero no pudo leer Secrets ni eliminar pods. Tampoco pudimos escribir en la raíz del contenedor. Este Deployment usa emptyDir. El disco LUKS lo probamos en la VM.

## 3. AgroSugar

Aplica `reto3-agrosugar/k8s/01-escenario.yaml` y crea el Secret db-cred en agrosugar con una contraseña sintética generada localmente, como indica el paso 6.1. Espera a que los pods estén listos. El archivo de políticas contiene cuatro documentos YAML. Para ver qué cambia en cada etapa, ejecuta la matriz primero sin políticas, luego con la primera y finalmente con las cuatro.

```bash
kubectl -n agrosugar wait --for=condition=Ready pods --all --timeout=180s
bash reto3-agrosugar/scripts/matriz.sh
# Aplicar solo el primer documento YAML (default-deny-all).
awk '/^---[[:space:]]*$/{exit} {print}' reto3-agrosugar/k8s/02-networkpolicies.yaml | kubectl apply -f -
bash reto3-agrosugar/scripts/matriz.sh
kubectl apply -f reto3-agrosugar/k8s/02-networkpolicies.yaml
kubectl get networkpolicy -n agrosugar
bash reto3-agrosugar/scripts/matriz.sh
```

Al principio pasaron las cuatro conexiones. Con la primera política quedaron todas bloqueadas. Con las cuatro políticas activas, solo el sensor pudo llegar al broker por TCP 1883. Para el paso 6.6, abre el suscriptor antes de publicar el mensaje. En nuestra prueba recibimos humedad_suelo con el valor 41.5 en agrosugar/lote-7/hidratacion.

## Auditoría final del historial

Para repetir el paso 7.1, abre Bash en la raíz de un clon con el historial completo. Haz el escaneo después de incorporar los cambios de entrega y guarda el identificador del commit junto con el resultado.

```bash
git rev-parse --short HEAD
docker run --rm -v "$PWD:/repo:ro" -w /repo zricethezav/gitleaks:latest detect --source /repo -v --redact
resultado=$?
echo "Código de salida de Gitleaks: $resultado"
git ls-files | grep -E '\.(key|keyfile|bak|img)$' || echo "Sin archivos rastreados con esas extensiones"
```

En el laboratorio, Gitleaks revisó cuatro commits y mostró `leaks found: 1`. Revisamos la línea marcada y vimos que correspondía a `encryptedData` del SealedSecret. En la línea 9, POSTGRES_PASSWORD estaba bajo spec.encryptedData. Por eso interpretamos esa alerta como un falso positivo sobre el valor cifrado. La consulta con git ls-files confirmó que Git no seguía archivos con las extensiones .key, .keyfile, .bak o .img.

Al repetirlo, comprueba el resumen y el código de salida. Si termina limpio, verás `no leaks found` y código 0. Si vuelve a señalar el valor cifrado, revisa la línea antes de documentar una excepción específica. Si encuentras una credencial en claro, rótala y sigue el paso 7.1 para sanear el historial. El último comando solo comprueba las extensiones de los archivos que Git sigue actualmente.

