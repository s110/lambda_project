# Scraper Serverless de Datos Inmobiliarios en AWS

Este proyecto implementa un sistema de web scraping automatizado, escalable y eficiente en costos para extraer datos de propiedades inmobiliarias del sitio web `infocasas.com.pe`. La arquitectura está completamente basada en servicios serverless de AWS, lo que garantiza un bajo mantenimiento y un modelo de pago por uso.

## 🎯 Objetivo del Proyecto

El objetivo principal es recopilar, enriquecer y almacenar información detallada de propiedades en venta (precio, ubicación, características, coordenadas geográficas, etc.) de forma periódica y automática. Los datos recopilados se almacenan en una base de datos NoSQL (Amazon DynamoDB), dejándolos listos para ser consumidos por otros servicios, como análisis de datos, APIs o paneles de visualización.

## 🏛️ Arquitectura de la Solución

La solución utiliza un patrón de diseño **"Fan-Out"** para desacoplar las tareas y permitir un procesamiento paralelo y resiliente. Esto evita los timeouts de las funciones Lambda y distribuye la carga de trabajo de manera eficiente.


### Flujo de Trabajo:

1.  **Disparo Programado**: Un disparador de **Amazon EventBridge** se activa según una programación definida (p. ej., una vez al día).
2.  **Orquestación**: El disparador invoca una función **AWS Lambda ("Orchestrator")**. Esta función es responsable de:
    *   Navegar por las páginas de listado del sitio web.
    *   Extraer los datos básicos de cada propiedad.
    *   Enviar cada propiedad como un mensaje individual a una cola de **Amazon SQS**.
3.  **Cola de Tareas**: La cola de **Amazon SQS** actúa como un búfer, almacenando de forma fiable todas las "tareas" (propiedades a procesar) pendientes.
4.  **Procesamiento en Paralelo**: La cola SQS activa una segunda función **AWS Lambda ("Worker")** por cada lote de mensajes. Gracias a la escalabilidad de Lambda, múltiples workers pueden ejecutarse en paralelo. Cada worker:
    *   Recibe un mensaje con los datos de una propiedad.
    *   Realiza una segunda petición al sitio web para visitar la página de detalle y extraer información adicional (en este caso, las coordenadas geográficas).
    *   Limpia y formatea el registro completo para que sea compatible con la base de datos.
5.  **Almacenamiento**: El worker guarda el registro final y enriquecido como un ítem en una tabla de **Amazon DynamoDB**.

