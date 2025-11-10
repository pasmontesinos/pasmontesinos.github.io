# Hugo + PaperMod bilingual starter (es/en)

Blog técnico con soporte para español e inglés, configurado para desplegar en GitHub Pages mediante GitHub Actions.

## 🚀 Pasos para configurarlo

1. **Crea el repositorio**
   - En GitHub, crea un repo llamado `blog` (puede ser privado).

2. **Sube este contenido**
   - Descarga este ZIP, descomprímelo y súbelo al repo.
   - O clónalo y haz push manualmente.

3. **Añade el tema PaperMod**
   ```bash
   git submodule add https://github.com/adityatelange/hugo-PaperMod themes/PaperMod
   git commit -m "add PaperMod theme"
   ```

4. **Activa GitHub Pages**
   - En tu repo → Settings → Pages → Source → selecciona **GitHub Actions**.

5. **Publica**
   - Haz commit y push a `main`.
   - GitHub Actions construirá el sitio y lo publicará en tu URL:
     - `https://<tu-usuario>.github.io/blog/`

6. **Edita los archivos**
   - Tus posts en `content/posts/`: usa sufijos `.es.md` y `.en.md` para el idioma.
   - Tu página personal: `content/about.es.md` y `content/about.en.md`.
   - Página principal: `content/_index.es.md` y `content/_index.en.md`.
   - Ajusta `baseURL` y `title` en `hugo.toml`.

Listo 🎉