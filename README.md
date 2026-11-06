# test-manager-library-3239188-api

> **Prueba técnica · Ficha ADSO 3239188**
> Horario: **jueves 5 de noviembre de 2026, de 7:00 p. m. a 11:00 p. m.** (hora Colombia). Entrega a las **11:00 p. m.**

Este repositorio es el **backend** de *Manager Library*, en **PHP con Laravel**. **Empieza vacío a propósito**: se construye en el fork de cada aprendiz.

## Instrucciones

Cada aprendiz debe **crear el fork** de los seis repositorios del proyecto y **resolver el proyecto
con el spec planteado**.

1. Hacer fork, a su cuenta de GitHub, de cada repositorio de la tabla del final.
2. Leer el spec en [`test-manager-library-3239188-docs`](https://github.com/code-sena/test-manager-library-3239188-docs).
   Se entrega en dos versiones: `spec-python/` y `spec-.net/`.
3. Desarrollar en los forks.

## El reto se desarrolla con React y PHP (Laravel)

El spec está escrito para Python y para .NET, pero el reto **no** se hace en esos lenguajes:

| Capa | Tecnología del reto |
|---|---|
| Frontend | React |
| Backend | PHP con Laravel |

Lo que el spec define sobre el negocio —historias, criterios de aceptación, reglas, contrato de la
API, modelo de datos— se respeta. Lo que define sobre la tecnología se traduce a React y Laravel.

## La prueba no consiste en escribir el código

El propósito principal es ver la **capacidad de desempeño con SDD** (*Spec-Driven Development*,
desarrollo guiado por especificación): cómo se lee, se interpreta y se aplica una especificación
para llevarla a un stack distinto. El código es el medio, no el fin.

## Entrega

Cuenta lo que quede **subido a los forks antes de las 11:00 p. m.** Lo que llegue después no se revisa.
No hay prórroga: suban avances parciales con commits pequeños durante la prueba.

## Los seis repositorios

| Repositorio | Qué va ahí |
|---|---|
| [`test-manager-library-3239188-docs`](https://github.com/code-sena/test-manager-library-3239188-docs) | El spec: `spec-python/` y `spec-.net/` |
| [`test-manager-library-3239188-api`](https://github.com/code-sena/test-manager-library-3239188-api) | Backend en PHP (Laravel) |
| [`test-manager-library-3239188-app`](https://github.com/code-sena/test-manager-library-3239188-app) | Frontend en React |
| [`test-manager-library-3239188-page`](https://github.com/code-sena/test-manager-library-3239188-page) | Sitio público estático de presentación |
| [`test-manager-library-3239188-infra`](https://github.com/code-sena/test-manager-library-3239188-infra) | Contenedores, red, volúmenes y motor de base de datos vacío |
| [`test-manager-library-3239188-tool`](https://github.com/code-sena/test-manager-library-3239188-tool) | Utilidades: sembrador de datos de demostración |

## Contenido de este repositorio

| Documento | Qué es |
|---|---|
| DECISIONES.md | Traducción del spec .NET a Laravel |
| docs/migraciones.md | Modelo de datos y orden de migraciones |
| docs/endpoints.md | Contrato de la API, roles y errores |
| docs/trazabilidad.md | Matriz HU → CA → endpoint → tarea → evidencia |

Alcance: se entrega el diseño completo (documentación guiada por el spec).