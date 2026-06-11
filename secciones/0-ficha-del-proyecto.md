### **0.1. Tu nombre completo:**
Mauricio Solano

### **0.2. Nombre del proyecto:**
ShopGuard — Sistema de Videovigilancia Inteligente con IA para Retail

### **0.3. Descripción breve del proyecto:**

ShopGuard es un sistema de videovigilancia inteligente que detecta comportamientos
sospechosos de robo en tiempo real dentro de comercios. Procesa el vídeo de una o varias
cámaras (USB o IP/RTSP), aplica modelos de visión por computador (YOLOv8 + MediaPipe Pose)
sobre cada frame y, cuando reconoce un patrón de riesgo, genera una alerta y notifica al
responsable por Telegram adjuntando la foto del frame sospechoso.

**Fase actual de desarrollo:** El sistema funciona para una tienda con su dashboard local
(FastAPI + Streamlit) y dispone de un gateway central que agrega N tiendas detrás de una
sola URL con su propia SPA y autenticación. La detección corre 100% en local, sin servicios
de pago en la nube.

### **0.4. URL del proyecto:**
https://github.com/msolano/ShopGuard

### **0.5. URL o archivo comprimido del repositorio:**
https://github.com/msolano/ShopGuard
