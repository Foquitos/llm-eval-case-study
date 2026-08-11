# Cómo medir si un auditor de calidad basado en LLM realmente acierta

**Case study — diseño del sistema de evaluación de una plataforma de auditoría con IA en producción**

Ignacio Otranto · AI Engineer
[linkedin.com/in/ignacio-julian-otranto](https://linkedin.com/in/ignacio-julian-otranto)

> Este documento describe el **diseño y el razonamiento** detrás de un sistema de evaluación que
> construí para una plataforma interna de auditoría de calidad. No incluye código propietario,
> datos de clientes ni resultados comerciales. Los ejemplos numéricos son ilustrativos y sirven
> para explicar el problema metodológico, no para reportar métricas de negocio.

---

## El contexto

En un BPO / contact center, el área de Calidad escucha llamadas y evalúa a los operadores contra
una rúbrica: ¿saludó según protocolo?, ¿verificó identidad?, ¿ofreció el canal de autogestión?,
¿cometió algún error crítico? Históricamente eso se hace a mano sobre una muestra: entre el 2 % y
el 5 % de las interacciones.

Reemplazar esa escucha por un LLM es, en apariencia, un problema resuelto: se transcribe el audio,
se le pasa la rúbrica al modelo y se le pide una respuesta por atributo. El prototipo funciona en
una tarde y las respuestas *parecen* buenas.

El problema real aparece después, cuando alguien de Calidad pregunta lo único que importa:

> **¿Cuánto le podemos creer a esto?**

Y ahí no alcanza con que las respuestas parezcan buenas. Ese fue el sistema que tuve que diseñar.

---

## Por qué el porcentaje de acierto engaña

La primera métrica que todo el mundo pide es el porcentaje de coincidencia entre la IA y un humano.
Es también la que más rápido lleva a una conclusión falsa.

En una campaña sana, la enorme mayoría de los atributos da **OK**: el operador saludó, verificó,
se despidió. Supongamos un 90 % de OK. Un prompt que respondiera **siempre "OK"**, sin leer nada,
sacaría **90 % de accuracy**. Y sería completamente inútil: jamás detectaría un error del operador,
que es exactamente para lo que existe el sistema.

Peor todavía: ese 90 % le da a Calidad una falsa sensación de seguridad, y a quien ajusta el prompt
una métrica que no se mueve cuando el prompt mejora ni cuando empeora.

### Kappa de Cohen

La métrica principal del sistema es el **kappa de Cohen**, que mide el acuerdo entre dos
evaluadores **descontando el acuerdo que se explica por el azar**. Un kappa de 0 significa "acuerda
tanto como una moneda cargada con la misma distribución"; 1, acuerdo perfecto.

La consecuencia práctica es el diagnóstico que ninguna otra métrica da:

> Un atributo con **92 % de accuracy y kappa 0,05** es un atributo que **no está midiendo nada**.
> Coincide porque casi todo es OK, no porque entienda el criterio.

El kappa se calcula **por atributo**, no por plantilla. Es la única granularidad accionable: una
rúbrica de 35 atributos no está "buena" o "mala" en bloque — típicamente hay 30 que andan bien y
3 o 4 que arrastran el resto y que hay que reescribir.

---

## Los errores no son simétricos

La segunda decisión de diseño es que la matriz de confusión no se colapsa en un único número,
porque para el negocio los errores **cuestan distinto**:

| Error | Qué pasa | Costo |
|---|---|---|
| **Falso error crítico** | La IA marca un error grave donde el humano no lo ve | Pone la llamada en 0 y castiga a un operador que no se equivocó. **El error más caro del sistema.** |
| **Error crítico omitido** | Se deja pasar un error grave real | El sistema no cumple su función, pero no daña a nadie de forma directa. |
| **Sin responder** | La IA se escapa por la salida de emergencia ("N/A") donde sí había evidencia | No rompe a nadie, pero vacía la auditoría y le devuelve el trabajo a Calidad. |

El tercero tiene una sutileza que costó ver: **un mismo comportamiento se escribe distinto según el
tipo de atributo**. Un atributo obligatorio responde `N/A`; uno opcional simplemente omite el campo.
Si no se normalizan a la misma clase, la métrica termina dependiendo de con qué tipo de dato se
modeló el criterio, que es una decisión de implementación sin ningún significado para el negocio.

---

## Golden Sets: contra qué se mide

Un evaluador necesita verdad de referencia. La construcción de esa verdad tiene tres decisiones que
resultaron ser las que más impacto tuvieron:

**1. La revisión humana no pisa la auditoría publicada.** Cuando un analista corrige a la IA, la
corrección se guarda **aparte**. Es contraintuitivo —lo natural es "arreglar" el registro— pero si
se sobrescribe, se destruye el dato que se quiere medir: desaparece qué había respondido el modelo.

**2. Las muestras son estratificadas y están congeladas.** Cada Golden Set se arma por plantilla,
estratificado por resultado (OK / NO OK / error crítico) para que las clases raras —las únicas
interesantes— estén representadas, y dividido en `train` / `test`. Además, sus audios quedan
**fijados contra el descarte automático** del almacenamiento: el store borra por antigüedad, así que
sin ese pin el set de referencia se evapora solo a las pocas semanas. Detalle de infraestructura,
consecuencia metodológica total.

**3. El techo lo pone el acuerdo entre humanos.** El sistema también mide el acuerdo **entre dos
analistas** sobre los mismos casos. Ese número es el techo real: **lo que dos personas expertas no
logran acordar entre ellas no se le puede exigir al modelo**. Sin esa referencia, se termina
persiguiendo un kappa de 0,9 en un criterio en el que los humanos apenas llegan a 0,6, y culpando
al prompt de una ambigüedad que está en la rúbrica.

---

## Versionar el prompt como se versiona el código

Cada auditoría y cada revisión humana quedan atadas a la **versión exacta del prompt que las
produjo**, identificada por hash del contenido. Eso habilita cuatro cosas que sin versionado son
imposibles:

- **Comparar prompts entre sí sobre la misma verdad humana**, sin gastar un token: la verdad ya
  está en la base, los resultados de cada versión también.
- **Detectar regresiones antes de publicar** un cambio de prompt.
- **Rollback** a una versión anterior cuando un ajuste empeora las cosas.
- **Vigencia**: cuando el cliente cambia un criterio, la verdad humana de los atributos afectados
  queda vieja. El sistema marca **esos atributos**, no el set entero — invalidar todo obligaría a
  rehacer meses de revisión por un cambio en un solo punto de la rúbrica.

---

## Dos caminos, según el costo

La evaluación se expone de dos formas, y la división es deliberada:

**Pantalla web (para Calidad).** Kappa por atributo, matriz de confusión, falsos errores críticos,
desvío de puntaje. Sale **todo de SQL sobre datos ya calculados: cero tokens, respuesta inmediata**.
La usa gente no técnica, todos los días.

**Evaluador de consola (para mí).** Lo mismo, más `--replay`: re-audita el Golden Set completo con
la plantilla actual para comparar antes/después de tocar un prompt. Tarda minutos y consume tokens,
y por eso **no** está en la pantalla: si estuviera, alguien lo dispararía sin querer.

Es una decisión de producto tanto como de ingeniería. La métrica barata tiene que estar siempre
disponible; la cara tiene que costar un acto deliberado.

---

## Lo que me llevo

1. **Elegir la métrica es una decisión de diseño, no un trámite final.** Accuracy sobre clases
   desbalanceadas no mide calidad: mide el desbalanceo.
2. **Desagregar por atributo y por tipo de error.** Un número global esconde exactamente los casos
   que importan.
3. **La verdad de referencia hay que protegerla**: de la sobreescritura bienintencionada, del
   descarte automático del storage y del paso del tiempo cuando cambian los criterios.
4. **Medir primero el techo humano.** Sin él no hay forma de saber si el problema es el modelo o
   la rúbrica.
5. **Versionar prompts vale tanto como versionar código**, y por la misma razón: sin eso no se
   puede afirmar que un cambio mejoró algo.
6. **Separar lo barato de lo caro.** Una métrica que cuesta tokens y minutos no puede vivir en la
   misma pantalla que una que sale de una consulta SQL.

---

## Stack

Python · FastAPI · SQL Server · LLMs (Gemini, Anthropic) · LlamaIndex · Qdrant ·
`pytest` (el módulo de métricas es puro: sin base de datos, sin IO y sin tokens, testeado end-to-end)
