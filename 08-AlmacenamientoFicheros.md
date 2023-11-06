- [Almacenamiento de ficheros](#almacenamiento-de-ficheros)
  - [Añadiendo subida de ficheros a nuestros endpoints](#añadiendo-subida-de-ficheros-a-nuestros-endpoints)
  - [Configurando el sistema de almacenamiento.](#configurando-el-sistema-de-almacenamiento)
- [Práctica de clase, Ficheros e imágenes](#práctica-de-clase-ficheros-e-imágenes)
- [Proyecto del curso](#proyecto-del-curso)

![](images/banner08.png)

# Almacenamiento de ficheros
Vamos a ver cómo subir ficheros a nuestra aplicación, siguiendo las estas [indicaciones](https://spring.io/guides/gs/uploading-files/). 

Para ello vamos a implementar la interfaz `StorageService` que nos permitirá guardar los ficheros en el sistema de archivos. 

Lo primero es indicar la carpeta donde queremos que se almacene la información. En nuestro fichero de configuración properties creamos la clave `upload.root-location` con el valor del directorio donde queremos guardar la información. Es en esta carpeta será donde se guarden los ficheros que subamos a través de nuestro servicio web. 

Para el almacenamiento en disco implementaremos la interfaz `StorageService` como `FileSystemStorageService`. Para ello, debemos crear la clase `FileSystemStorageService` que implemente la interfaz `StorageService`. De esta manera si quisiésemos crear un almacenamiento en la nube solo tendríamos que implementar `StorageService` con los métodos que necesitemos para el manejo de la nube o almacenamiento remoto. 

Es importante destacar los métodos:

- loadAsResource: Devuelve un objeto de tipo Resource a partir de un nombre de fichero y con ello podemos acceder a las propiedades de este.
- getUrl: Devuelve un objeto de tipo String con la URL del fichero a partir de su nombre. Puede er interesante si queremos enlace directo a nuestra imagen.


```java
/**
 * Método que es capaz de cargar un fichero a partir de su nombre
 * Devuelve un objeto de tipo Resource
 */
@Override
public Resource loadAsResource(String filename) {
    try {
        Path file = load(filename);
        Resource resource = new UrlResource(file.toUri());
        if (resource.exists() || resource.isReadable()) {
            return resource;
        } else {
            throw new StorageNotFoundException("No se puede leer fichero: " + filename);
        }
    } catch (MalformedURLException e) {
        throw new StorageNotFoundException("No se puede leer fichero: " + filename + " " + e);
    }
}

/**
 * Método que devuelve la URL de un fichero a partir de su nombre
 * Devuelve un objeto de tipo String
 */
@Override
public String getUrl(String filename) {
    return MvcUriComponentsBuilder
            // El segundo argumento es necesario solo cuando queremos obtener la imagen
            // En este caso tan solo necesitamos obtener la URL
            .fromMethodName(FilesController.class, "serveFile", filename, null)
            .build().toUriString();
}
```

Por otro lado, debemos hacer un controlador que nos permita gestionar las peticiones al Servicio de Almacenamiento, por ejemplo con un método GET para nuestro fichero o uno POST para subir ficheros, aunque este último podemos hacerlo dentro del controlador o enpoint del recurso.

En el método de subida usaremos la opción `MediaType.MULTIPART_FORM_DATA_VALUE` para que sea capaz de recibir un fichero.

Además en nuestro controlador, donde queramos subir un fichero, debemos añadir el parámetro `@RequestPart("file") MultipartFile file` para que Spring sea capaz de inyectar el fichero que queremos subir.

```java
@RestController
@RequestMapping("/api/files")
public class FilesController {
    private StorageService storageService;

    // También podemos inyectar dependencias por el setter
    @Autowired
    public void setStorageService(StorageService storageService) {
        this.storageService = storageService;
    }

    @GetMapping(value = "{filename:.+}")
    @ResponseBody
    public ResponseEntity<Resource> serveFile(@PathVariable String filename, HttpServletRequest request) {
        Resource file = storageService.loadAsResource(filename);

        String contentType = null;
        try {
            contentType = request.getServletContext().getMimeType(file.getFile().getAbsolutePath());
        } catch (IOException ex) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "No se puede determinar el tipo de fichero");
        }

        if (contentType == null) {
            contentType = "application/octet-stream";
        }

        return ResponseEntity.ok()
                .contentType(MediaType.parseMediaType(contentType))
                .body(file);
    }

    @PostMapping(value = "", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    // Aunque no es obligatorio, podemos indicar que se consume multipart/form-data
    // Para ficheros usamos, Resuqest part, porque lo tenemos dividido en partes
    public ResponseEntity<Map<String, Object>> uploadFile(
            @RequestPart("file") MultipartFile file) {

        // Almacenamos el fichero y obtenemos su URL
        String urlImagen = null;

        if (!file.isEmpty()) {
            String imagen = storageService.store(file);
            urlImagen = storageService.getUrl(imagen);
            Map<String, Object> response = Map.of("url", urlImagen);
            return ResponseEntity.status(HttpStatus.CREATED).body(response);
        } else {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "No se puede subir un fichero vacío");
        }
    }

    // Implementar el resto de metodos del servicio que nos interesen...
    // Delete file, listar ficheros, etc....
}
```

## Añadiendo subida de ficheros a nuestros endpoints
Ahora que tenemos la subida de ficheros implementada, es importante que en nuestros modelos tengamos un campo para almacenar el fichero asociado, o simplemente 

Lo primero es que en la anotación le vamos a indicar que consume `MediaType.MULTIPART_FORM_DATA_VALUE` para que sea capaz de recibir un fichero.

Además en nuestro controlador, donde queramos subir un fichero, debemos añadir el parámetro `@RequestPart("file") MultipartFile file` para que Spring sea capaz de inyectar el fichero que queremos subir.

Por ejemplo, en nuestro modelo de Raquetas, añadimos el campo `imagen` y lo podemos actualizar de la siguiente manera:

```java
// PATCH: /api/raquetas//imagen/{id}
// consumes = MediaType.MULTIPART_FORM_DATA_VALUE: Indica que el parámetro de la función es un parámetro del cuerpo de la petición HTTP
// @PathVariable: Indica que el parámetro de la función es un parámetro de la URL en este caso {id}
// @RequestPart: Indica que el parámetro de la función es un parámetro del cuerpo de la petición HTTP
// En este caso es un fichero, por lo que se indica con @RequestPart y mMltipartFile
@PatchMapping(value = "/imagen/{id}", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<RaquetaResponseDto> nuevoProducto(
        @PathVariable Long id,
        @RequestPart("file") MultipartFile file) {

    log.info("patchRaqueta");

    // Buscamos la raqueta
    if (!file.isEmpty()) {
        String imagen = storageService.store(file);
        String urlImagen = storageService.getUrl(imagen);

        var raqueta = raquetasService.findById(id);
        raqueta.setImagen(urlImagen);

        // Devolvemos el OK
        return ResponseEntity.ok(
                raquetaMapper.toResponse(raquetasService.update(id, raqueta))
        );
    } else {
        throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "No se ha enviado la imagen");
    }
}
```

Recomiendo llevar el el servicio fuera del controlador, para que esté integrado con el servicio que maneje el endpoint. Por otro lado, ten en cuneta que si actualizas una nueva imagen debes eliminar la antigua o mejor aún sobre-escribirla, y que si eliminas el item debes eliminar la imagen asociada si la tiene.

## Configurando el sistema de almacenamiento.
Otra de las cosas que debemos hacer es configurar el sistema de de almacenamiento. Esto es, crear el directorio si no esta creado y borrar las imágenes que hay si estamos en modo desarrollo.

Para esta labor vamos a hacer uso de una clase configurada con `@Config`. Esta anotación, hace que se cargue antes que el resto por el sistema de inversión de control. Y es una anotación específica para indicar que estas clases tienen elementos de configuración.

Además usaremos `CommandLineRunner` es una interfaz proporcionada por Spring Boot que te permite ejecutar código después de que el contexto de la aplicación se haya cargado y antes de que la aplicación se inicie. Proporciona un único método `run()` que acepta un arreglo de cadenas como argumento, el cual se puede utilizar para pasar argumentos de línea de comandos a la aplicación.

En el fragmento de código proporcionado, el método `init()` está anotado con `@Bean` para indicar que debe ser gestionado como un bean por el contenedor de Spring. El tipo de retorno del método init() es CommandLineRunner, lo que indica que este bean se ejecutará cuando la aplicación se inicie.

El propósito del método `init()` es inicializar el servicio de almacenamiento. Recibe una instancia de `StorageService` y un valor de la propiedad `upload.delete` como argumentos. La implementación del método verifica si la propiedad deleteAll está configurada como "true", y si es así, registra un mensaje en el registro y llama al método `deleteAll()` en storageService para eliminar todos los archivos. Finalmente, llama al método `init()` en storageService para realizar cualquier otra inicialización necesaria.

```java
@Configuration
@Slf4j
public class StorageConfig {
    @Bean
    public CommandLineRunner init(StorageService storageService, @Value("${upload.delete}") String deleteAll) {
        return args -> {
            // Inicializamos el servicio de ficheros
            // Leemos de application.properties si necesitamos borrar todo o no

            if (deleteAll.equals("true")) {
                log.info("Borrando ficheros de almacenamiento...");
                storageService.deleteAll();
            }

            storageService.init(); // inicializamos
        };
    }
}
```

Una forma alternativa de lograr el mismo resultado sin usar CommandLineRunner es utilizar la anotación `@PostConstruct` en un método dentro de un bean. La anotación @PostConstruct es una anotación estándar de Java que indica que un método debe ejecutarse después de que el bean haya sido construido y se hayan inyectado todas las dependencias.

En este ejemplo, el método init() está anotado con @PostConstruct, lo que indica que se debe ejecutar después de que el bean se haya construido y se hayan inyectado todas las dependencias. Dentro del método, se realiza la misma lógica que en el código original utilizando CommandLineRunner.

Elige el método que más se adapte a tus necesidades.
```java
@Configuration
@Slf4j
public class StorageConfig {
    @Autowired
    private StorageService storageService;
    
    @Value("${upload.delete}")
    private String deleteAll;
    
    @PostConstruct
    public void init() {
        if (deleteAll.equals("true")) {
            log.info("Borrando ficheros de almacenamiento...");
            storageService.deleteAll();
        }

        storageService.init(); // inicializamos
    }
}
```

# Práctica de clase, Ficheros e imágenes

1. Crea la carpeta funkos-images, y añade la funcionalidad para crear un servicio de almacenamiento y con ello poder cambiar/almacenar las imágenes a los funkos.
2. Testea los repositorios, servicios y controladores con la nueva funcionalidad.

# Proyecto del curso
Puedes encontrar el proyecto con lo visto hasta este punto en la etiqueta: [v.0.0.3 del repositorio del curso: ficheros](https://github.com/joseluisgs/DesarrolloWebEntornosServidor-02-Proyecto-2023-2024/releases/tag/ficheros).
