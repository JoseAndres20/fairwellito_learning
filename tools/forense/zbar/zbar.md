Para scanear QR

`zbar` sirve para leer códigos desde la terminal:
- **QR codes**
- **Códigos de barras** (EAN, UPC, Code128, etc.)
- **Data Matrix**
- **PDF417**
  
```bash
# Instalar si no lo tenés
sudo apt install zbar-tools

# Escanear el QR
zbarimg imagen.png
```