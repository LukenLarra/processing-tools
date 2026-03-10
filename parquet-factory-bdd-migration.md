# BDD Framework Migration — parquet-factory

## Contexto

Este documento describe los cambios realizados en `parquet-factory` para ejecutar los tests BDD a través del workflow centralizado de `processing-tools` y la GitHub Action **`RedHat-BDD-Framework`**.

---

## Qué había antes

El workflow llamaba al workflow de `processing-tools` en su rama `master`, que ejecutaba los tests dentro del contenedor `insights-behavioral-spec:latest`.

---

## Qué hay ahora

El workflow sigue siendo igual de simple — delega completamente en `processing-tools`. El único cambio es la referencia a la nueva rama que incorpora el framework.

```yaml
# ANTES
uses: RedHatInsights/processing-tools/.github/workflows/bdd.yaml@master

# AHORA
uses: LukenLarra/processing-tools/.github/workflows/bdd.yaml@bdd-framework-integration
```

### Workflow completo

```yaml
name: BDD tests

on:
  push:
    branches: ["main"]
  pull_request:
  workflow_dispatch:

jobs:
  bdd:
    name: BDD tests
    uses: LukenLarra/processing-tools/.github/workflows/bdd.yaml@bdd-framework-integration
    with:
      service: parquet-factory
      use_kafka: true
      use_minio: true
      use_pushgateway: true
```

---

## Lo que hace `processing-tools` por este repo

Al llamar al workflow de `processing-tools`, este se encarga automáticamente de:

- Levantar los servicios necesarios (Kafka, MinIO, Pushgateway) con la configuración correcta
- Clonar `insights-behavioral-spec` con los tests y steps
- Compilar el binario de `parquet-factory`
- Ejecutar los tests via la Action `RedHat-BDD-Framework`
- Publicar los resultados en la pestaña **Checks** del PR

No es necesario definir nada de esto en el repo de `parquet-factory`.
