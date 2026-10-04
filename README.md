# Ecosistema de Automatización IA: Triage de soporte

Entrega final de **Valentino Verdichio**.

Armé un sistema que recibe mails de soporte, los guarda en una base de datos, los analiza con IA y prepara una respuesta. Antes de contestarle al cliente, una persona tiene que aprobar el borrador.

El caso de uso es **Fennec**, una empresa ficticia de hosting web que recibe consultas de sus clientes por mail.

## Herramientas

| Parte | Herramienta | Para qué la uso |
|---|---|---|
| Orquestador | **n8n** | Conecta todo y tiene la lógica del flujo |
| Base de datos | **Airtable** | Guarda clientes, tickets y errores |
| IA | **Claude Haiku 4.5** (Anthropic) | Clasifica el ticket y escribe el borrador |
| Canal | **Gmail** | Recibe los mails, pide la aprobación y responde al cliente |

## Archivos del repositorio

| Archivo | Qué es |
|---|---|
| `Diagrama_Arquitectura.pdf` | Diagrama del flujo y explicación paso a paso |
| `workflow_ecosistema_ia.json` | El flujo de n8n (se puede importar) |
| `capturas/` | Evidencia de las pruebas |

## Links

- **Base de datos (solo lectura):** https://airtable.com/appIkxzSD9TzBvlFy/shrzlgDC5BSZL0f0s
- **Video demo:** _(agregar link)_

## Cómo funciona

1. **Llega un mail** a la casilla de soporte. n8n revisa Gmail cada minuto y solo trae mails nuevos y no leídos.
2. **Revisa si ya lo procesó:** busca el ID del mail en Airtable. Si ya existe lo ignora, así no se repite ni se generan bucles.
3. **Crea el ticket** en Airtable con estado `Pendiente`, vinculado al cliente que lo mandó.
4. **Valida el mensaje:** si tiene menos de 15 caracteres no se lo manda a la IA (para no gastar) y se guarda como error.
5. **Claude lo analiza** y devuelve categoría, prioridad del 1 al 5, sentimiento, resumen y un borrador de respuesta.
6. **Pide aprobación:** le manda el borrador por mail al aprobador con los botones *Aprobar* y *Rechazar*. El flujo se queda esperando la respuesta.
7. **Responde al cliente** en el mismo hilo del mail original. Si el borrador se rechaza, no se manda nada.
8. **Si algo falla**, el error se guarda en la tabla `Errores`, vinculado a su ticket.

### Base de datos

- **Clientes**: nombre, mail, empresa y plan.
- **Tickets**: cada mail recibido, con lo que dijo la IA y su estado.
- **Errores**: lo que salió mal, en qué nodo y en qué ticket.

Estados de un ticket: `Pendiente` → `Procesado por IA` → `Aprobado por Humano` → `Respondido` (o `Rechazado` / `Error IA`).

## Detalles técnicos que pedía la consigna

- **Trigger:** Gmail Trigger con filtro de búsqueda y solo mails no leídos. En modo activo solo toma mails nuevos.
- **IA:** nodo de Anthropic con `maxTokens: 600` y `temperature: 0.2`. La respuesta se lee de `content[].text` (el equivalente a `Message.Content`).
- **Prompt dinámico:** usa el nombre, la empresa y el plan del cliente, y el asunto y el texto del mail.
- **Manejo de errores:**
  - *Resume:* los nodos de Claude, del parseo de la respuesta y del envío por Gmail tienen salida de error. Si fallan, el flujo sigue y guarda el error en Airtable.
  - *Break:* un Error Trigger registra cualquier fallo que no estaba previsto.
- **Human-in-the-loop:** el nodo de Gmail "Send and Wait" frena el flujo hasta que alguien aprueba o rechaza.
- **Thread ID:** la respuesta se manda como *reply* del mail original, así queda en la misma conversación.

## Pruebas

| # | Qué probé | Resultado |
|---|---|---|
| 1 | Cliente registrado con un problema urgente | ✅ Llegó como URGENTE, lo aprobé y se respondió en el hilo |
| 2 | Rechazar el borrador | ✅ Ticket en `Rechazado`, el cliente no recibió nada |
| 3 | Mail que solo dice "hola" | ✅ No pasó por la IA y se guardó el error |
| 4 | Mail de alguien que no es cliente | ✅ Se creó el ticket sin cliente vinculado |
| 5 | El mismo mail procesado dos veces | ✅ Lo detectó y lo ignoró |
| 6 | Falla de la IA (puse un modelo que no existe) | ✅ Se guardó el error y el ticket quedó en `Error IA` |
| 7 | Respuesta de la IA cortada (bajé Max Tokens a 20) | ✅ No pudo leer el JSON, se guardó el error y el ticket quedó en `Error IA` |
| 8 | Falla al enviar la respuesta por Gmail | ✅ Se guardó el error y el ticket quedó en `Aprobado por Humano` (aprobado pero sin enviar) |

## Check de seguridad

- **¿Hay un filtro contra bucles infinitos?** Sí. Se ignoran los mails que manda la propia casilla y se busca el ID del mail en Airtable antes de procesarlo.
- **¿Los filtros comparan tipos correctos?** Sí. El largo del mensaje (≥ 15) y la prioridad (≥ 4) se comparan número contra número.
- **¿El prompt es dinámico?** Sí, se arma con los datos de cada cliente y de cada mail.

## Capturas

### Flujo en n8n
![Flujo completo](capturas/01_flujo_n8n_completo.png)
![Ejecuciones](capturas/02_ejecuciones_n8n.png)

### Prueba 1: camino feliz
![Ejecución camino feliz](capturas/03_prueba1_camino_feliz.png)
![Mail de aprobación](capturas/04_mail_aprobacion.png)
![Respuesta en el mismo hilo](capturas/05_respuesta_en_hilo.png)

### Prueba 2: rechazo
![Ticket rechazado](capturas/06_prueba2_rechazado.png)

### Prueba 3: datos incompletos
![Datos incompletos](capturas/07_prueba3_datos_incompletos.png)

### Prueba 5: mail duplicado
![Duplicado](capturas/08_prueba5_duplicado.png)

### Prueba 6: falla de la IA
![Error IA](capturas/09_prueba6_error_ia.png)

### Prueba 7: respuesta de la IA cortada
![Respuesta inválida](capturas/13_prueba7_respuesta_invalida.png)

### Prueba 8: falla del envío
![Error de envío](capturas/14_prueba8_error_envio.png)

### Base de datos
![Tickets](capturas/10_airtable_tickets.png)
![Errores](capturas/11_airtable_errores.png)
![Clientes](capturas/12_airtable_clientes.png)

