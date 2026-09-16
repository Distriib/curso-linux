# Curso RHEL — PGN Panamá

Material para un curso de 40 horas (RH124 + RH134) que el usuario dicta en vivo.
Es pentester con Linux fuerte, pero **nuevo en RHEL específicamente** — viene de
otras distros y de un perfil de seguridad ofensiva, no de administración RHEL.

## Regla explícita: corregime si me equivoco

Si en algún momento afirmo algo técnicamente incorrecto, impreciso o
desactualizado sobre Linux, RHEL, redes, comandos o cualquier tema del
curso — **corregilo de inmediato**, aunque:
- suene seguro de lo que digo,
- esté en medio de otra explicación,
- ya lo haya dicho antes sin que se me corrigiera.

Esto no es opcional ni cosmético: lo que se habla acá se enseña en vivo a un
salón de estudiantes minutos después. Un error mío sin corregir se convierte
en información falsa que un grupo entero aprende mal. Ante la duda, señalá la
duda en vez de dejarla pasar — es preferible una corrección de más que un dato
falso enseñado en clase.

## Convenciones de este repo

- `temario/dia-XX/dia-XX.md` — guía completa **para el instructor**: incluye
  notas de clase, "si preguntan por...", horarios pedagógicos, soluciones de
  retos y todo lo que no debe llegar a los estudiantes.
- `temario/dia-XX/summary.md` — agenda resumida del día (horarios y bloques),
  también material del instructor.
- Versiones **para estudiantes**: mismo contenido, sin nada de lo anterior,
  reescrito en tono directo (sin "si preguntan por", sin notas de manejo de
  grupo, sin comparar "instructor vs. participantes" — todo en segunda
  persona neutral). Se generan como archivos aparte (sufijo `-estudiantes.md`
  o carpeta `estudiantes/`, según lo que se acuerde en cada día) y se
  exportan a PDF para repartir.
- Conversión markdown → PDF: no hay pandoc/LibreOffice instalados; se usa un
  script Python con `markdown` + `xhtml2pdf` (ambos instalables con `pip3
  install markdown xhtml2pdf`, sin dependencias nativas). Para los `.pptx`
  (láminas de conceptos), exportar a PDF vía AppleScript con Microsoft
  PowerPoint (`osascript`, comando `save ... as save as PDF`) — el export
  normal ya excluye las notas del orador, que viven aparte en cada slide.
