# Informe técnico - Caracterización del modelo

| Dato | Cómo obtenerlo | Valor |
|---|---|---|
| Perfil de hardware | Sección 2 de la guía | B (8 GB) |
| RAM total del equipo | `free -h` | 4.0 GB visibles en WSL (equipo de 16 GB) |
| Modelo y etiqueta | `ollama list` | qwen2.5:0.5b |
| Tamaño en disco | `ollama list` | 397 MB |
| Latencia de 5 ejecuciones (ms) | `time curl ...` cinco veces | 5099, 3871, 3136, 3297, 3339|
| Latencia promedio | Promedio de las cinco | 3748 ms |
| RAM usada durante la inferencia | `free -h` mientras responde | 1001 MB |
| Calidad percibida (1 a 5) | Su criterio, con una frase que lo justifique | 3: responde en español y acierta lo básico, pero inventa datos y no respeta instrucciones como responder en una sola frase |
