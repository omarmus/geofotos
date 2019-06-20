# geo-fotos

## Instalación para producción

### Compilar y minificar para producción
En el archivo `vue.config.js` colocar el subdominio si se require en la propiedad `publicPath` luego ejecutar el comando:

```
npm run build
```

Copiar el contenido de la carpeta `dist` en la carpeta `/var/www/html` o donde esté redireccionando el Nginx

**Nota.-** Este frontend require del backend que está en la carpeta `server` de este proyecto.

## Instalación para desarrollo
```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

### Run your tests
```
npm run test
```

### Lints and fixes files
```
npm run lint
```

### Customize configuration
See [Configuration Reference](https://cli.vuejs.org/config/).
