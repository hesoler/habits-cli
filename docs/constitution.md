# Constitución — habits-cli

## Principios innegociables

1. **Stack mínimo.** Solo stdlib + pytest. Nueva dependencia solo con aprobación explícita y justificación.
2. **Spec antes que código.** Cada funcionalidad se describe en un spec en `specs/` antes de implementarse.
3. **Lógica ≠ interfaz.** Núcleo puro en `habits/core.py`, CLI en `habits/cli.py`. La CLI llama al núcleo; el núcleo no conoce la CLI.
4. **Tests obligatorios.** Todo cambio funcional incluye tests. Sin tests no hay merge.
5. **Persistencia local.** Datos en `habits_data.json` en la raíz del proyecto (ignorado por Git), con esquema JSON fijo: `{"habits": [{"name": str, "created": str, "completed_dates": list}]}`.
6. **Inglés en código, español en mensajes.** Identificadores y comentarios en inglés; textos que ve el usuario en español.
