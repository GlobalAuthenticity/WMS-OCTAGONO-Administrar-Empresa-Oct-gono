# Gestión de Empresas

**Versión:** 1.0.0 · **Autor:** Cristian · **Requiere:** WordPress 5.8+ · PHP 7.4+ · **Text domain:** `gestion-empresas`

> Lista las empresas (rol group_admin) con sus áreas y las elimina en cascada, con usuarios y datos, dentro de una transacción SQL.

## ¿Qué es?

Es un plugin interno de administración que trabaja sobre las **empresas** del sistema, es decir, los usuarios con el rol **`group_admin`**. Permite tres cosas:

1. **Ver** todas las empresas con su número de áreas (registros de `{prefix}areas` con `grupo_id` igual al ID de la empresa).
2. **Buscar** empresas en tiempo real, ordenar la lista y paginarla.
3. **Eliminar** una empresa **por completo**: sus áreas, todos los usuarios inscritos en esas áreas y los datos de esos usuarios en WordPress, LearnDash, WatuPRO, WooCommerce, Ultimate Member y Elementor. Todo se hace dentro de **una sola transacción SQL**: o se borra todo o no se borra nada.

La pantalla está en el menú **Empresas** de wp-admin y solo pueden usarla los usuarios con `manage_options`.

---

## Funcionalidades

### 1. Listado de empresas
- Tabla basada en `WP_List_Table` con tres columnas: **Empresa** (nombre, correo e ID), **Áreas** y **Acciones**.
- Se puede ordenar por **ID, nombre, correo o número de áreas**. Para ordenar por áreas, el plugin calcula los conteos y ordena la lista en PHP, porque `WP_User_Query` no puede consultar `{prefix}areas`.
- Muestra 20 empresas por página.
- Los conteos de áreas se calculan con **una sola consulta** para toda la página (`GROUP BY grupo_id`).

### 2. Búsqueda en tiempo real
- Mientras se escribe, el plugin espera un momento y busca por AJAX (`ge_search_companies`), sin recargar la página.
- Busca en `display_name`, `user_email`, `user_login` y `user_nicename`.
- El servidor devuelve la tabla ya generada en HTML, con su orden y paginación.

### 3. Eliminación en cascada
Desde el menú **Acciones** de cada fila:

1. **Confirmación** con SweetAlert2, que advierte de que la acción es irreversible y muestra el nombre de la empresa.
2. **Overlay y barra de progreso**: bloquean la página durante el borrado para evitar clics repetidos.
3. **Resultado**: un mensaje de éxito o de error y la tabla actualizada.

#### Orden del borrado (`GE_Deletion_Service`)
```
START TRANSACTION
  1. IDs de las áreas de la empresa  ←  {prefix}areas WHERE grupo_id = empresa
  2. IDs de los usuarios de esas áreas  ←  {prefix}user_members WHERE area_id IN (…)
  3. Borrado completo de cada uno de esos usuarios
  4. Borrado de las inscripciones en user_members y de las áreas de la empresa
     → do_action('ge_before_company_commit', …)
  5. Borrado completo del propio usuario group_admin
COMMIT   (si algo falla: ROLLBACK, sin cambios)
```

#### Datos que se borran de cada usuario
Solo se tocan las tablas que existen en la instalación; para comprobarlo se usa `SHOW TABLES LIKE`.

| Origen | Tablas |
|---|---|
| WordPress | `usermeta`, `comments`, `users` |
| Sistema de áreas | `user_members` |
| Ultimate Member | `um_metadata` |
| LearnDash | `learndash_user_activity` (+ `_meta`), `ld_achievements`, `learndash_pro_quiz_lock`, `learndash_pro_quiz_statistic_ref`, `learndash_pro_quiz_toplist` |
| WatuPRO / Watu | `watupro_payments`, `watupro_student_answers`, `watupro_taken_exams`, `watupro_unlock_logs`, `watupro_users_groups`, `watupro_user_certificates`, `watupro_user_files`, `watu_takings` |
| WooCommerce | `woocommerce_payment_tokens` (+ `tokenmeta`), `woocommerce_downloadable_product_permissions`, `wc_download_log`, `wc_customer_lookup`, `wc_order_product_lookup`, `wc_order_stats` |
| Elementor | `e_notes_users_relations`, `e_submissions` |

- Los **pedidos de WooCommerce no se borran** a propósito, porque puede haber obligación contable o legal de conservarlos. Solo se borran las tablas de análisis (*lookup*) y los tokens de pago guardados.

---

## Hooks para ampliar el borrado

| Hook | Cuándo se ejecuta | Parámetros |
|---|---|---|
| `ge_delete_user_related_data` | Por cada usuario, antes de borrarlo de `users` y dentro de la transacción | `$user_id`, `$wpdb` |
| `ge_before_company_commit` | Antes de borrar al `group_admin` y de hacer el COMMIT | `$company_id`, `$area_ids`, `$member_user_ids` |

Se usan para borrar datos de tablas propias o de otros plugins dentro de la misma transacción.

## Seguridad
- Los dos endpoints AJAX (`ge_search_companies` y `ge_delete_company`) exigen `manage_options` y el nonce `ge_ajax_nonce`.
- Antes de borrar, se comprueba que el ID corresponde a un usuario con el rol `group_admin`.
- Todas las consultas usan `$wpdb->prepare()`. Los nombres de tabla y de columna están fijos en el código; ninguno viene de lo que envía el usuario.
- Si falla cualquier consulta, se lanza `GE_Deletion_Exception` y se hace **ROLLBACK**.

## Arquitectura
```
gestion-empresas/
├── gestion-empresas.php                       # Bootstrap y constantes (GE_COMPANY_ROLE = group_admin)
├── includes/
│   ├── class-ge-plugin.php                    # Registro de hooks
│   ├── class-ge-admin-page.php                # Menú, página, assets y textos para el JS
│   ├── class-ge-list-table.php                # WP_List_Table (Empresa / Áreas / Acciones)
│   ├── class-ge-ajax-controller.php           # Endpoints AJAX
│   └── services/
│       ├── class-ge-company-service.php       # Lectura: listado, búsqueda, conteo de áreas
│       └── class-ge-deletion-service.php      # Borrado en cascada dentro de una transacción
└── assets/
    ├── css/admin.css
    └── js/admin.js                            # Búsqueda, menú de acciones, SweetAlert2 y progreso
```

## Dependencias
- Tablas `{prefix}areas` y `{prefix}user_members`.
- SweetAlert2 11.10.1, que se carga desde cdnjs.
- **Motor InnoDB** en las tablas: la transacción solo garantiza el borrado de todo o nada con InnoDB.

## Limitaciones y riesgos conocidos
- **Usuarios en varias empresas:** se borra **por completo** a cualquier usuario inscrito en un área de la empresa, aunque también pertenezca a áreas de otras empresas.
- **No usa `wp_delete_user()`:** borra directamente en la base de datos. Por eso:
  - No se ejecutan los hooks `delete_user` ni `deleted_user`.
  - No se limpia la caché de usuarios.
  - Las entradas escritas por esos usuarios no se reasignan ni se borran.
  - Queda `commentmeta` huérfano.
- **Posible borrado de datos de otros clientes en WooCommerce:** `wc_order_product_lookup` y `wc_order_stats` se borran usando el `user_id` como `customer_id`. En WooCommerce, `customer_id` es el ID interno de `wc_customer_lookup`, que no es el mismo número que el `user_id`. Esto puede borrar datos de análisis de otro cliente cuyo `customer_id` coincida por casualidad con ese número.
- **La barra de progreso es simulada:** la transacción es una sola petición, así que la barra avanza a ritmo fijo hasta que llega la respuesta del servidor.
