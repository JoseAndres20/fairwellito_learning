# 🧠 fairwellito_learning

**Repositorio público** de documentación de aprendizaje en **Ciberseguridad**.

---

## 🎯 Propósito

Este repositorio es mi cuaderno digital donde documento todo lo que voy aprendiendo en ciberseguridad de forma práctica y ordenada:

- 🧪 **Laboratorios:** Ejercicios prácticos de TryHackMe, HackTheBox, CTFs y más
- 🛠️ **Herramientas:** Guías, comandos y cheatsheets de herramientas de ciberseguridad
- 📸 Evidencias y capturas (sin datos sensibles)

**Finalidad:** Aprendizaje educativo y ético. Todo el contenido es con fines legítimos de estudio y práctica autorizada.

---

## ⚠️ Aviso Importante

Este repositorio es **público** y de **solo lectura** para visitantes.

- ❌ **NO** contiene ni debe contener credenciales, claves, contraseñas o datos sensibles
- ✅ Todo el contenido es educativo y con fines éticos
- ✅ Los laboratorios documentados se realizan en entornos controlados y autorizados

---

## 📋 Cómo Hacer la Documentación

Para mantener consistencia y profesionalismo, toda la documentación sigue estas reglas:

### ✅ Formato Markdown (.md)
- Un archivo `.md` por tema/laboratorio
- Nombres descriptivos: `ssh-basics.md`, `nmap-scanning.md`, `thm-linux-priv-esc.md`
- Estructura clara con títulos jerárquicos (`#`, `##`, `###`)

### 📐 Estructura de Documentos
```markdown
# Título Principal

## Objetivo / Introducción
Qué se va a aprender o hacer.

## Herramientas / Requisitos
Lista de lo necesario.

## Pasos / Desarrollo
Proceso detallado con comandos y explicaciones.

## Conclusión / Notas
Aprendizajes clave.

## Referencias
Enlaces útiles.
```

### 🎨 Buenas Prácticas
- **Espacio entre secciones:** Siempre una línea en blanco entre títulos, párrafos y bloques de código
- **Bloques de código con sintaxis:**
  ````markdown
  ```bash
  nmap -sV 192.168.1.1
  ```
  ````
- **Imágenes con texto alternativo:**
  ```markdown
  ![Captura de Nmap](./assets/nmap-scan.png)
  ```
- **Listas claras y consistentes:**
  ```markdown
  - Item 1
  - Item 2
  - Item 3
  ```

📖 **Guía completa:** Consulta [`markdown-best-practices.md`](./markdown-best-practices.md) para más detalles.

---

## 📂 Estructura del Repositorio

```
fairwellito_learning/
├── README.md                       # Este archivo
├── laboratorios/                   # Labs, CTFs, ejercicios prácticos
│   ├── thm-linux-basics.md
│   ├── htb-weak-rsa.md
│   └── overthewire-bandit.md
└── herramientas/                   # Guías de herramientas y comandos
    ├── nmap-cheatsheet.md
    ├── metasploit-basics.md
    └── burpsuite-guide.md
```

---

## 🚀 Cómo Usar Este Repo

### Clonar el repositorio
```bash
git clone https://github.com/JoseAndres20/fairwellito_learning.git
cd fairwellito_learning
```

### Crear un nuevo documento
```bash
# Crear laboratorio
nano laboratorios/2025-10-mi-nuevo-lab.md

# Crear guía de herramienta
nano herramientas/nmap-guide.md
```

### Guardar cambios
```bash
git add .
git commit -m "docs: agregar laboratorio de X"
git push origin main
```

---

## 🧭 Navegación

- 🧪 **Laboratorios:** [`/laboratorios`](./laboratorios)
- 🛠️ **Herramientas:** [`/herramientas`](./herramientas)

---

## 👨‍💻 Autor

**José Andrés Acuña Rodríguez** — _"fairwellito"_  
Estudiante de Ingeniería en TI – UTN Costa Rica

Documentación en Markdown — Material de apoyo y aprendizaje en ciberseguridad.

---

## 📜 Licencia

Este repositorio es de carácter educativo y personal. El contenido es libre para consulta y referencia con fines de aprendizaje.

---

**Última actualización:** Octubre 2025
