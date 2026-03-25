# 🚀 LA GUÍA DEFINITIVA Y MASIVA: BACKEND DELIVERUS (NIVEL DIOS)

Esta guía contiene todas las herramientas, métodos y patrones necesarios para resolver los exámenes de Node.js, Express y Sequelize (tipo Reviews, Schedules, Orders, etc.).

---

## 1. ANATOMÍA DE LA PETICIÓN Y RESPUESTA (Express)

### A. Lo que recibimos (`req`)
* 🌐 **`req.params` (La URL):** Para identificar el recurso exacto. Ej: `/restaurants/:restaurantId` -> `req.params.restaurantId`.
* 📦 **`req.body` (El Paquete / JSON):** Los datos que envía el usuario (formularios). **¡Siempre hay que validarlos!**
* 🛡️ **`req.user` (La Identidad Segura):** Creado por el servidor al leer el Token. Contiene `req.user.id`. NUNCA confíes en un ID de usuario que venga en el `body`.
* 🔍 **`req.query` (Filtros extra):** Parámetros opcionales en la URL tras el símbolo `?`. Ej: `/restaurants?status=online` -> `req.query.status`.

### B. Lo que devolvemos (`res`)
* **`res.json(datos)`:** Envía un objeto o array en formato JSON (el 99% de las veces usarás esto si todo va bien).
* **`res.status(codigo).send(mensaje)`:** Para enviar errores o mensajes de texto plano.
* **`res.status(codigo).json({ error: '...' })`:** Para enviar errores estructurados.

---

## 2. DICCIONARIO DE CÓDIGOS HTTP (El idioma del Frontend)
Si devuelves el código equivocado, los tests fallarán. Úsalos sabiamente en tus `catch` y middlewares.

* **✅ 200 OK:** Todo perfecto (lecturas, borrados, actualizaciones).
* **✅ 201 Created:** Recurso creado con éxito (a veces el test acepta 200 en su lugar, pero 201 es más pro).
* **❌ 400 Bad Request:** El cliente envió datos basura (esto lo devuelve automáticamente `express-validator` si falla).
* **🔒 401 Unauthorized:** No hay token o ha caducado (el usuario no ha hecho login).
* **⛔ 403 Forbidden:** Está logueado, pero no tiene permiso. Ej: Un cliente intentando borrar un restaurante, o alguien intentando reseñar sin haber pedido.
* **👻 404 Not Found:** El recurso no existe (Ej. `checkEntityExists`).
* **⚠️ 409 Conflict:** Regla de negocio rota. Ej: "Ya has reseñado este restaurante" o "Los horarios se solapan".
* **💥 500 Internal Server Error:** Explotó la base de datos (va en los `catch (err)`).

---

## 3. EL ARSENAL DE SEQUELIZE (Consultas a BD)

No solo existe `findAll` y `findByPk`. Aquí tienes todas las herramientas para extraer datos.

### A. Métodos de Búsqueda
* **`Model.findByPk(id)`:** Busca un solo registro por su Clave Primaria.
* **`Model.findOne({ where: {...} })`:** Devuelve el PRIMER registro que cumpla la condición. Útil para ver si existe algo específico.
* **`Model.findAll({ where: {...} })`:** Devuelve un ARRAY con todos los que cumplan la condición.
* **`Model.count({ where: {...} })`:** Devuelve un NÚMERO entero. **¡Súper eficiente para middlewares!** No trae los datos, solo los cuenta.

### B. Operadores Avanzados (`[Op]`)
Si necesitas buscar por rangos, fechas o textos parciales, necesitas importar los Operadores:
`import { Op } from 'sequelize'`

* **`[Op.gt] / [Op.gte]`:** Mayor que / Mayor o igual que (Greater Than / Equal).
* **`[Op.lt] / [Op.lte]`:** Menor que / Menor o igual que (Less Than / Equal).
* **`[Op.ne]`:** Distinto de (Not Equal).
* **`[Op.or] / [Op.and]`:** Múltiples condiciones complejas.

**Ejemplo de uso de Operadores (Schedules o Precios):**
```javascript
const horariosTarde = await Schedule.findAll({
  where: {
    restaurantId: 1,
    openingHour: { [Op.gte]: '16:00' } // Trae los que abren a las 16:00 o más tarde
  }
})
```

### C. Métodos de Modificación
* **Crear:** `const obj = Model.build(req.body); obj.miId = 1; await obj.save()` (Recomendado para inyectar IDs de forma segura).
* **Crear Directo:** `await Model.create({ campo: 'valor' })`.
* **Actualizar:** `await Model.update(req.body, { where: { id: req.params.id } })`.
* **Borrar:** `await Model.destroy({ where: { id: req.params.id } })`.

---

## 4. EXPRESS-VALIDATOR (Protección Máxima)

Todos los métodos para limpiar y asegurar `req.body` antes de que llegue al controlador.

### A. Presencia y Ausencia
* `check('campo').exists()` -> Tiene que venir en el JSON.
* `check('campo').notEmpty()` -> No puede ser un string vacío `""`.
* `check('campo').optional({ nullable: true, checkFalsy: true })` -> Puede faltar, ser null o estar vacío.
* `check('campo').not().exists()` -> **¡PROHIBIDO!** (Úsalo siempre para `restaurantId`, `userId`, etc. que inyectas tú).

### B. Tipos de Datos y Rangos
* **Strings:** `.isString()`, `.isLength({ min: 3, max: 255 })`.
* **Números:** `.isInt({ min: 1, max: 5 })`, `.isFloat({ min: 0 })`.
* **Listas/Arrays:** `.isArray()`.
* **Booleanos:** `.isBoolean()`.
* **Fechas/Horas:** `.isDate()`, `.isTime({ hourFormat: 'hour24' })`.
* **Valores cerrados (Enum):** `.isIn(['pending', 'delivered', 'cancelled'])`.

### C. Transformadores (Sanitizers - Limpian los datos)
Siempre van al final de la cadena de validación.
* `.toInt()` / `.toFloat()` -> Convierte strings numéricos ("5") a números reales (5).
* `.trim()` -> Quita espacios en blanco a los lados de un texto.
* `.escape()` -> Convierte caracteres HTML peligrosos (seguridad anti-XSS).

### D. Validadores Custom (La navaja suiza)
Si necesitas comparar dos campos del mismo `body` (Ej. horarios) o comprobar la BD:
```javascript
check('closingHour').custom((value, { req }) => {
  if (value <= req.body.openingHour) {
    throw new Error('La hora de cierre debe ser posterior a la de apertura');
  }
  return true;
})
```

---

## 5. MIDDLEWARES CUSTOM (Reglas de Negocio)

Si te piden "comprobar que no existe solapamiento", "comprobar que ha hecho un pedido" o "comprobar que no ha reseñado ya", hazlo aquí.

**Estructura Universal de un Middleware de Negocio:**
```javascript
const checkReglaDeNegocio = async (req, res, next) => {
  try {
    // 1. Buscamos o Contamos en la Base de Datos
    const cantidad = await Modelo.count({
      where: { 
        usuarioId: req.user.id, // Sacado del token
        restauranteId: req.params.restaurantId // Sacado de la URL
        // ... otras condiciones (status, fechas, etc)
      }
    })
    
    // 2. Comprobamos la regla
    if (cantidad > 0) {
      // 3A. Si rompe la regla, lo echamos con un 409 o 403
      return res.status(409).send('Regla de negocio rota. No puedes pasar.')
    }
    
    // 3B. Si cumple la regla, le dejamos seguir al Controlador
    return next()
    
  } catch (err) {
    res.status(500).send(err)
  }
}
```

---

## 6. PLANTILLAS INFALIBLES PARA CONTROLADORES (CRUD)

### 📖 GET (Index - Listar de un padre)
```javascript
async index (req, res) {
  try {
    const records = await Model.findAll({
      where: { parentId: req.params.parentId },
      // attributes: { exclude: ['password'] }, // Opcional: Ocultar columnas
      // order: [['createdAt', 'DESC']]         // Opcional: Ordenar
    })
    res.json(records)
  } catch (err) { res.status(500).send(err) }
}
```

### ✍️ POST (Create - Inyección Segura)
```javascript
async create (req, res) {
  try {
    const newRecord = Model.build(req.body)
    
    // 🛡️ INYECCIÓN SEGURA DE IDs:
    newRecord.userId = req.user.id              // De quién es (Token)
    newRecord.restaurantId = req.params.restaurantId // Dónde se crea (URL)
    
    const savedRecord = await newRecord.save()
    res.json(savedRecord)
  } catch (err) { res.status(500).send(err) }
}
```

### ♻️ PUT/PATCH (Update)
```javascript
async update (req, res) {
  try {
    await Model.update(req.body, { where: { id: req.params.id } })
    const updatedRecord = await Model.findByPk(req.params.id)
    res.json(updatedRecord)
  } catch (err) { res.status(500).send(err) }
}
```

### 🗑️ DELETE (Destroy)
```javascript
async destroy (req, res) {
  try {
    const result = await Model.destroy({ where: { id: req.params.id } })
    if (result === 1) {
      res.json('Borrado con éxito.')
    } else {
      res.status(404).json('No se pudo borrar, no existe.')
    }
  } catch (err) { res.status(500).send(err) }
}
```

---

## 7. ORDEN LÓGICO DE LAS RUTAS (`routes.js`)
Recuerda, los middlewares se ejecutan de Izquierda a Derecha. El controlador SIEMPRE va al final.

```javascript
app.route('/padre/:padreId/hijo')
  .post(
    isLoggedIn,                            // 1. Seguridad Token
    hasRole('customer'),                   // 2. Rol permitido
    checkEntityExists(Padre, 'padreId'),   // 3. Verifica si el Padre (Restaurante) existe (Da 404)
    MiMiddleware.comprobarRegla,           // 4. Reglas de negocio (Da 403 o 409)
    MisValidaciones.create,                // 5. Verifica req.body
    handleValidation,                      // 6. Expulsa si falló el paso 5 (Da 400)
    MiControlador.create                   // 7. Guarda en BD (Da 200/201)
  )
```

# 🚀 GUÍA RÁPIDA DE SUPERVIVENCIA: EXAMEN DELIVERUS

## 1. EL ORIGEN DE LOS DATOS (`req.params`, `req.body`, `req.user`)
Saber de dónde viene cada dato es vital para la seguridad y el funcionamiento de tu API.

* 🌐 **`req.params` (La URL)**
    * **Qué es:** Datos visibles en la barra de direcciones (ej. `/restaurants/:restaurantId`).
    * **Para qué se usa:** Para identificar el recurso exacto sobre el que estamos actuando (ej. `req.params.restaurantId` o `req.params.reviewId`).
* 📦 **`req.body` (El Paquete / JSON)**
    * **Qué es:** La información oculta que envía el frontend (formularios).
    * **Para qué se usa:** Para los datos editables (texto, puntuaciones, nombres). ¡Es lo que validamos con `express-validator`!
* 🛡️ **`req.user` (La Identidad Segura)**
    * **Qué es:** Información generada por tu servidor leyendo el Token del usuario. ¡Nunca viene del cliente!
    * **Para qué se usa:** Para saber **quién** hace la acción (ej. `req.user.id`).
    * **Regla de oro:** JAMÁS saques el ID del creador/dueño del `req.body`. Siempre usa `req.user.id`.

---

## 2. VALIDACIONES (`express-validator`)
Son el escudo de tu base de datos. Se definen antes de llegar al controlador.

* **Validación Básica (Obligatorio):**
    ```javascript
    check('stars').exists().isInt({ min: 0, max: 5 }).toInt()
    // ¡Ojo! No olvides los paréntesis en .toInt()
    ```
* **Validación Opcional (Campos no obligatorios):**
    ```javascript
    check('body').optional({ nullable: true, checkFalsy: true }).isString().trim()
    ```
    * *`nullable: true`* -> Acepta que envíen `"body": null`.
    * *`checkFalsy: true`* -> Acepta que envíen strings vacíos `"body": ""`.
* **Prohibir campos por seguridad (MUY IMPORTANTE):**
    ```javascript
    check('customerId').not().exists(),   // Lo asignará el servidor (req.user.id)
    check('restaurantId').not().exists()  // Lo sacaremos de la URL (req.params)
    ```

## 2. VALIDACIONES SUPER VITAMINADAS (`express-validator`)
Son el escudo de tu base de datos. Se definen antes de llegar al controlador para asegurar que el `req.body` trae justo lo que necesitas.

### A. Existencia y Opcionalidad
* **Obligatorio (que venga en el body y no sea nulo):**
  `check('campo').exists({ checkNull: true })`
* **Que no esté vacío (ej. que no sea un string de espacios):**
  `check('campo').notEmpty()`
* **Opcional (si no viene, no pasa nada):**
  `check('campo').optional({ nullable: true, checkFalsy: true })`
* **Prohibido (para IDs que sacamos del token o de la URL):**
  `check('campo').not().exists()`

### B. Tipos de Datos (Strings, Números, Booleanos)
* **Textos (Cadenas):**
  `check('nombre').isString()`
* **Números Enteros:**
  `check('edad').isInt()`
* **Números Decimales (Precios, etc.):**
  `check('precio').isFloat()`
* **Booleanos (True / False):**
  `check('estaActivo').isBoolean()`
* **Fechas:**
  `check('fechaNacimiento').isDate()`
* **Arrays (Listas):**
  `check('listaProductos').isArray()`

### C. Reglas y Límites (Rangos, Tamaños, Opciones)
* **Rango de números (Mínimo y Máximo):**
  `check('estrellas').isInt({ min: 0, max: 5 })`
* **Longitud de un texto (Ej. contraseñas o descripciones):**
  `check('password').isLength({ min: 8, max: 255 })`
* **Opciones cerradas (Enum / Solo aceptar ciertos valores):**
  `check('estadoPedido').isIn(['pending', 'delivered', 'cancelled'])`
* **Formato de Email:**
  `check('correo').isEmail()`
* **Formato de URL (Enlaces web o imágenes):**
  `check('foto').isURL()`
* **Expresiones regulares (Avanzado, ej. formato de DNI o teléfono):**
  `check('telefono').matches(/^[0-9]{9}$/)`

### D. Transformadores (Sanitizers)
¡Ojo! Estos no validan, **modifican** el dato que te llega para limpiarlo antes de guardarlo. Siempre se ponen al final de la cadena.
* **Convertir a Entero (Super importante para IDs o contadores):**
  `.toInt()`
* **Convertir a Decimal (Super importante para precios):**
  `.toFloat()`
* **Quitar espacios en blanco al principio y al final del texto:**
  `.trim()`
* **Convertir a booleano real:**
  `.toBoolean()`

### E. Ejemplo de Cadena Completa (Combo Múltiple)
* *Ejemplo: Un precio opcional, que si viene debe ser decimal, entre 0 y 1000, y lo convertimos.*
  ```javascript
  check('price')
    .optional({ nullable: true, checkFalsy: true })
    .isFloat({ min: 0, max: 1000 }).withMessage('El precio debe estar entre 0 y 1000')
    .toFloat()
  ```

### F. Validaciones Personalizadas (Custom validations)
Si necesitas hacer una comprobación en la base de datos (Ej. Comprobar que no estás metiendo un producto con un nombre que ya existe):
```javascript
check('name').custom(async (value) => {
  const productoExistente = await Product.findOne({ where: { name: value } })
  if (productoExistente) {
    throw new Error('El nombre de este producto ya existe')
  }
  return true
})
```
```

¡Con esta lista ya no habrá campo en el examen que se te resista! Tienes desde los tipos más básicos hasta las validaciones personalizadas. ¿Qué te parece?
---

## 3. DOMINANDO SEQUELIZE (Consultas a Base de Datos)
Las 4 herramientas clave para pedirle datos a Sequelize:

### A. `where` (El Filtro)
Sirve para decirle "búscame solo los que cumplan esta condición".
```javascript
where: { restaurantId: req.params.restaurantId }
```

### B. `include` (El JOIN / Traer a los amigos)
**⚠️ REGLA DE ORO:** Úsalo **SOLO** si necesitas los datos completos de otra tabla (ej. el nombre del restaurante, su dirección, etc.). Si solo necesitas el ID (`restaurantId`), **NO uses include**, Sequelize ya te da el ID automáticamente.
```javascript
// Ejemplo de cuándo SÍ usarlo: Traer el restaurante y además los detalles completos de su categoría.
include: { model: RestaurantCategory, as: 'restaurantCategory' }
```

### C. `attributes` (Elegir columnas)
Para ocultar campos sensibles (como contraseñas o IDs de usuario) de la respuesta que va al cliente.
```javascript
attributes: { exclude: ['userId'] }
```

### D. `order` (Ordenar resultados)
Para ordenar por alguna columna.
```javascript
order: [['stars', 'DESC']] // Ordena de mayor a menor puntuación
```

---

## 4. PLANTILLAS DE CONTROLADORES (CRUD)
Estructura perfecta para los métodos de tu controlador.

**🔍 INDEX (Listar filtrando por URL):**
```javascript
async index (req, res) {
  try {
    const reviews = await Review.findAll({
      where: { restaurantId: req.params.restaurantId }
    })
    res.json(reviews)
  } catch (err) { res.status(500).send(err) }
}
```

**➕ CREATE (Construir, inyectar IDs y guardar):**
```javascript
async create (req, res) {
  try {
    const newReview = Review.build(req.body) // Mete lo que envía el cliente
    
    // INYECCIÓN SEGURA DE IDs:
    newReview.customerId = req.user.id               // Quién lo hace
    newReview.restaurantId = req.params.restaurantId // Dónde lo hace
    
    const review = await newReview.save() // Guarda en BD
    res.json(review)
  } catch (err) { res.status(500).send(err) }
}
```

**🗑️ DESTROY (Borrar y comprobar si existía):**
```javascript
async destroy (req, res) {
  try {
    const result = await Review.destroy({ 
      where: { id: req.params.reviewId } 
    })
    if (result === 1) { // 1 significa que encontró y borró 1 fila
      res.json('Successfully deleted.')
    } else {
      res.json('Could not delete.')
    }
  } catch (err) { res.status(500).send(err) }
}
```

---

## 5. MIDDLEWARES DE REGLAS DE NEGOCIO
Cuando te pidan "Comprobar si el usuario ya ha hecho X antes de dejarle hacer Y". Usa `count()` o `findOne()`.

**Ejemplo: Comprobar que no existe ya una reseña de este usuario en este restaurante:**
```javascript
const checkCustomerHasNotReviewed = async (req, res, next) => {
  try {
    const count = await Review.count({
      where: { 
        customerId: req.user.id,               // 1. Este usuario...
        restaurantId: req.params.restaurantId  // 2. ...en este restaurante
      }
    })
    
    if (count === 0) return next() // Todo OK, pasa al controlador
    
    return res.status(409).send('Ya has hecho una reseña aquí.') // Bloqueado
  } catch (err) { res.status(500).send(err) }
}
```

---

## 6. 🔥 CHECKLIST DE ERRORES FRECUENTES (¡Léelo antes de entregar!)
1. **Nombres de las columnas:** Asegúrate de si la tabla usa `userId` o `customerId`. Míralo en los archivos del modelo o en el README.
2. **`req.user.id` vs `req.user.customerId`:** El usuario del token **SIEMPRE** se saca con `req.user.id`, sin importar cómo se llame la columna en la tabla de reviews.
3. **Parámetros mal escritos:** Cuidado al escribir `req.params.reviewId` (revisa cómo se llama el parámetro en el archivo de Rutas con los dos puntos `:reviewId`).
4. **Imports no usados (Clean Code):** Si tu controlador de `Review` no hace un `await Restaurant.findAll()`, borra `Restaurant` de la línea de `import` arriba del todo.
