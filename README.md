# Esau Martinez's Blog

Blog personal construido con [Hugo](https://gohugo.io/) y el tema [Stack](https://github.com/CaiJimmy/hugo-theme-stack).

## Prerrequisitos

- [Git](https://git-scm.com/)
- [Go](https://go.dev/dl/) (v1.17+)
- [Hugo extended](https://gohugo.io/installation/) (v0.120+)

## Setup local

```bash
# Clonar el repositorio
git clone https://github.com/dinoesau/blog.git
cd blog

# Inicializar módulos de Hugo
hugo mod tidy

# Iniciar servidor de desarrollo
hugo server
```

Abrir `http://localhost:1313` en el navegador.

## Comandos útiles

| Comando | Descripción |
|---|---|
| `hugo server` | Servidor de desarrollo con recarga automática |
| `hugo` | Build del sitio en `public/` |
| `hugo new content/en/post/mi-post/index.md` | Crear un post en inglés |
| `hugo new content/es/post/mi-post/index.md` | Crear un post en español |

## Idiomas

El sitio es bilingüe (inglés y español). El contenido se organiza por idioma:

- `content/en/` - contenido en inglés (idioma por defecto, servido en la raíz)
- `content/es/` - contenido en español (servido bajo `/es/`)

Los títulos, menús y parámetros por idioma se configuran en `config/_default/languages.toml`.

## Publicar un post

1. Crear el archivo del post.
2. Ejecutar `hugo server` para previsualizar en local.
3. Hacer commit y push a `main`. GitHub Actions despliega automáticamente a GitHub Pages.

## Actualizar el tema

```bash
hugo mod get -u github.com/CaiJimmy/hugo-theme-stack/v4
hugo mod tidy
```
