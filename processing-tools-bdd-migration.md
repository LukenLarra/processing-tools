# BDD Framework Migration — processing-tools

## Contexto

Este documento describe los cambios realizados en `processing-tools` para migrar la ejecución de tests BDD desde un modelo basado en **imagen Docker + scripts `.sh`** a un modelo basado en la GitHub Action **`RedHat-BDD-Framework`**.

El workflow de este repositorio actúa como orquestador central reutilizable. Cualquier servicio que quiera ejecutar sus tests BDD solo necesita llamar a este workflow con unos pocos parámetros.

---

## Qué había antes

- El job corría dentro de un contenedor `insights-behavioral-spec:latest`
- Los services (Kafka, MinIO, Pushgateway) no necesitaban `ports` mapping porque compartían red Docker con el contenedor
- La ejecución de tests se hacía via `cd ${BDD_PATH} && make ${{ inputs.service }}`
- Los logs se subían manualmente con `actions/upload-artifact`

```yaml
# Fragmento del workflow original
jobs:
  bdd:
    container:
      image: quay.io/redhat-services-prod/obsint-processing-tenant/insights-behavioral-spec/insights-behavioral-spec:latest
    services:
      kafka:
        image: quay.io/ccxdev/kafka-no-zk:latest  # sin ports, sin env
    steps:
      - name: Run BDD tests
        run: cd ${BDD_PATH} && make ${{ inputs.service }}
      - name: Store tests logs
        uses: actions/upload-artifact@v5
        with:
          path: ${{ env.BDD_PATH }}/logs
```

---

## Qué hay ahora

- Se elimina el `container` — el job corre directamente en `ubuntu-latest`
- Se añaden dos checkouts: el servicio bajo test en el workspace raíz e `insights-behavioral-spec` en subcarpeta
- Todos los services tienen `ports` mapping explícito para ser accesibles desde el runner via `localhost`
- La ejecución de tests se delega a la Action `RedHat-BDD-Framework`
- Los reportes se publican automáticamente por la Action en GitHub Checks (eliminando los steps manuales de logs)

---

## Cambios en los services

Todos los services necesitan `ports` mapping explícito para ser accesibles desde el runner via `localhost`. Kafka además requiere `KAFKA_ADVERTISED_HOST_NAME: localhost` y un health check.

```yaml
# ANTES
kafka:
  image: quay.io/ccxdev/kafka-no-zk:latest

# AHORA
kafka:
  image: ${{ inputs.use_kafka == true && 'quay.io/ccxdev/kafka-no-zk:latest' || '' }}
  ports:
    - 9092:9092
  env:
    KAFKA_ADVERTISED_HOST_NAME: localhost
  options: >-
    --health-cmd "kafka-topics.sh --bootstrap-server localhost:9092 --list"
    --health-interval 10s
    --health-timeout 10s
    --health-retries 10
```

El mismo patrón de `ports` se aplica al resto de services:

| Service | Puerto |
|---|---|
| `minio` | 9000 |
| `pushgateway` | 9091 |
| `database` | 5432 |
| `mock-oauth2-server` | 8081 |

---

## Steps añadidos

```yaml
- name: Checkout insights-behavioral-spec
  uses: actions/checkout@v4
  with:
    repository: LukenLarra/insights-behavioral-spec
    ref: bdd-framework-migration
    path: insights-behavioral-spec

- name: Install kcat
  if: ${{ inputs.use_kafka }}
  run: sudo apt-get install -y kafkacat

- name: Copy framework config
  run: cp insights-behavioral-spec/bdd-configs/${{ inputs.service }}-framework.yml framework.yml

- name: Add hosts entries
  if: ${{ inputs.use_pushgateway }}
  run: echo "127.0.0.1 pushgateway" | sudo tee -a /etc/hosts

- name: Run BDD Framework
  uses: LukenLarra/RedHat-BDD-Framework@main
  with:
    service: ${{ inputs.service }}
    bdd_config: "${{ github.workspace }}/framework.yml"
    test_requirements: "insights-behavioral-spec/requirements.txt"
```

---

## Decisiones de diseño

### Por qué se elimina el contenedor

El contenedor `insights-behavioral-spec:latest` actuaba como entorno de ejecución unificado donde todos los servicios compartían red y los paths eran fijos. Al eliminarlo, la Action `RedHat-BDD-Framework` toma ese rol, ganando transparencia, mantenibilidad y publicación automática de reportes en GitHub Checks.

### Por qué se copia el `framework.yml` al workspace raíz

`bdd_framework.py` resuelve los paths relativos usando como `root_path` el directorio donde está el fichero. Si el fichero está en `insights-behavioral-spec/bdd-configs/`, paths como `insights-behavioral-spec/features/parquet-factory` se resolverían incorrectamente. Copiándolo al workspace raíz, `root_path` es el workspace y todos los paths funcionan correctamente.

### Por qué `KAFKA_ADVERTISED_HOST_NAME: localhost`

La imagen `kafka-no-zk` se anuncia por defecto con el hostname `kafka` (como en docker-compose). En GHA el runner solo puede acceder al service via `localhost`, por lo que es necesario sobreescribir este valor.

### Por qué `/etc/hosts` para Pushgateway

Los feature files referencian Pushgateway como `pushgateway:9091`, hostname resoluble en docker-compose pero no en GHA. Añadir la entrada en `/etc/hosts` resuelve el problema sin modificar los feature files ni romper compatibilidad con el entorno local.
