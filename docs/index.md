# 📚 Documentación del Proyecto MVC Enlaces

## 🔄 Ciclo de Vida del Desarrollo de la Aplicación

A continuación, se describe el proceso completo de creación de esta aplicación MVC, desde la configuración de la base de datos hasta la implementación y pruebas.

---

## 🗄️ 1. Configuración de la Base de Datos

### Paso 1: Crear las Tablas en `phpMyAdmin`

1. Abre `phpMyAdmin` o cualquier herramienta para gestionar MySQL.
2. Ejecuta las siguientes consultas SQL para crear las tablas `categoria` y `vinculos`:

```sql
CREATE TABLE `categoria` (
    `pk_categoria` int NOT NULL AUTO_INCREMENT,
    `categoria` varchar(25) DEFAULT NULL,
    `tipo` varchar(15) DEFAULT NULL,
    PRIMARY KEY (`pk_categoria`),
    UNIQUE KEY `categoria` (`categoria`)
) ENGINE=InnoDB;

CREATE TABLE `vinculos` (
    `pk_vinculo` int NOT NULL AUTO_INCREMENT,
    `enlace` varchar(255) DEFAULT NULL,
    `titulo` varchar(255) DEFAULT NULL,
    `fk_categoria` int DEFAULT NULL,
    PRIMARY KEY (`pk_vinculo`),
    KEY `fk_vinculos_categorias` (`fk_categoria`),
    CONSTRAINT `fk_vinculos_categorias` FOREIGN KEY (`fk_categoria`) REFERENCES `categoria` (`pk_categoria`)
) ENGINE=InnoDB;
```
### Paso 2: Crear la Vista vista_enlaces
Esta vista combina las tablas categoria y vinculos para facilitar las búsquedas:

```sql
CREATE VIEW vista_enlaces AS
SELECT
    vinculos.pk_vinculo,
    vinculos.enlace,
    vinculos.titulo,
    categoria.categoria AS nombre_categoria,
    categoria.tipo AS tipo_categoria
FROM vinculos
JOIN categoria ON vinculos.fk_categoria = categoria.pk_categoria;
```

## 💻 2. Desarrollo del Objeto ModelBBDD
### Propósito:
El objeto ModelBBDD maneja la conexión a la base de datos y las consultas.

#### Código Completo:
```php
<?php

class ModelBBDD {
    private $host = 'localhost';
    private $dbname = 'enlaces';
    private $user = 'root';
    private $password = '';
    private $pdo;

    public function __construct() {
        try {
            $this->pdo = new PDO("mysql:host={$this->host};dbname={$this->dbname}", $this->user, $this->password);
            $this->pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        } catch (PDOException $e) {
            die("Error de conexión: " . $e->getMessage());
        }
    }

    public function getVistaEnlaces($query = "", $filterType = "") {
        $sql = "SELECT * FROM vista_enlaces";
        $params = [];

        if (!empty($filterType)) {
            switch ($filterType) {
                case 'categoria':
                    $sql .= " WHERE nombre_categoria LIKE ?";
                    $params[] = "%$query%";
                    break;
                case 'lenguaje':
                    $sql .= " WHERE tipo_categoria = 'LENGUAJE'";
                    if (!empty($query)) {
                        $sql .= " AND (nombre_categoria LIKE ? OR titulo LIKE ?)";
                        $params[] = "%$query%";
                        $params[] = "%$query%";
                    }
                    break;
                case 'titulo':
                    $sql .= " WHERE titulo LIKE ?";
                    $params[] = "%$query%";
                    break;
                default:
                    throw new Exception("Filtro no reconocido");
            }
        }

        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
}
```

## 🏗️ 3. Desarrollo de la Aplicación MVC
### Estructura del Proyecto
```plaintext
/project-root
├── /assets
│   └── styles.css            # Estilos personalizados
├── /controllers
│   ├── Autoload.php          # Carga automática de clases
│   ├── ViewController.php    # Gestión centralizada de vistas
│   ├── Router.php            # Sistema de enrutamiento
│   └── PreguntasController.php # Controlador para búsquedas
├── /models
│   └── ModelBBDD.php         # Clase para manejar la conexión a la base de datos
├── /views
│   ├── preguntas.php         # Formulario de búsqueda
│   ├── respuesta.php         # Resultados de búsqueda
│   ├── header.php            # Encabezado (Navbar)
│   └── footer.php            # Pie de página
└── index.php                 # Punto de entrada principal
```

---
### a) Archivo `Autoload.php`
Encargado de cargar automáticamente las clases necesarias.

```php
<?php

spl_autoload_register(function ($class) {
    $paths = ['./controllers/', './models/'];
    foreach ($paths as $path) {
        $file = $path . $class . '.php';
        if (file_exists($file)) {
            require_once $file;
            return;
        }
    }
    throw new Exception("Clase no encontrada: $class");
});
```
### b) Archivo `ViewController.php`
Gestiona la renderización de vistas.

```php
<?php

class ViewController {
    public static function render($view, $data = []) {
        extract($data); // Extrae las variables para usarlas en la vista.
        $viewPath = './views/' . $view . '.php';

        if (file_exists($viewPath)) {
            require './views/header.php';
            require $viewPath;
            require './views/footer.php';
        } else {
            throw new Exception("Vista no encontrada: $view");
        }
    }
}
```

### c) Archivo `PreguntasController.php`
Gestiona las búsquedas y renderiza las vistas.

```php
<?php

class PreguntasController {
    public function index() {
        ViewController::render('preguntas');
    }

    public function buscar() {
        $modelo = new ModelBBDD();
        $query = $_POST['query'] ?? '';
        $filter = $_POST['filter'] ?? '';

        if (empty($filter)) {
            $error = "Debe seleccionar un filtro antes de buscar.";
            ViewController::render('preguntas', ['error' => $error]);
            return;
        }

        $resultados = $modelo->getVistaEnlaces($query, $filter);
        ViewController::render('respuesta', ['resultados' => $resultados]);
    }
}
```

### ✅ 4. Pruebas y Resultados
Probar la Aplicación
1. Configura la base de datos como se indica.
2. Ejecuta el servidor local:
```bash
php -S localhost:8000
```
3. Abre en el navegador: http://localhost:8000.
