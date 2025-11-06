# Fix: Error de Descarga ERR_RESPONSE_HEADERS_MULTIPLE_CONTENT_DISPOSITION

## Problema Identificado

Al intentar descargar reportes de quiz en formatos Excel (.xlsx), CSV (.csv) u ODS (.ods) desde Moodle, se generaba el siguiente error en el navegador:

```
ERR_RESPONSE_HEADERS_MULTIPLE_CONTENT_DISPOSITION
```

**Error técnico**: El servidor enviaba múltiples encabezados HTTP `Content-Disposition`, lo cual viola el estándar HTTP y causa que los navegadores modernos (Chrome, Edge, Firefox) rechacen la descarga.

## Causa Raíz

La librería Spout (usada para generar archivos Excel/ODS/CSV) tiene un método `openToBrowser()` que envía automáticamente sus propios encabezados HTTP, incluyendo `Content-Disposition`. Esto causaba duplicación cuando Moodle u otros componentes ya habían enviado headers.

## Solución Implementada

### Archivo Modificado
`lib/classes/dataformat/spout_base.php`

### Cambios Realizados

#### 1. Limpieza de Output Buffers (líneas 68-71)
```php
// Clear all output buffers to prevent corruption
while (ob_get_level() > 0) {
    ob_end_clean();
}
```
**Propósito**: Eliminar cualquier contenido en buffer que pueda corromper la descarga.

#### 2. Envío Manual de Headers (líneas 82-101)
```php
// Send headers manually instead of using openToBrowser
if (is_https()) {
    header('Cache-Control: max-age=10');
    header('Pragma: ');
} else {
    header('Cache-Control: private, must-revalidate, pre-check=0, post-check=0, max-age=0');
    header('Pragma: no-cache');
}
header('Expires: ' . gmdate('D, d M Y H:i:s', 0) . ' GMT');

// Set content type based on format
if ($this->spouttype == \Box\Spout\Common\Type::XLSX) {
    header('Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet');
} else if ($this->spouttype == \Box\Spout\Common\Type::ODS) {
    header('Content-Type: application/vnd.oasis.opendocument.spreadsheet');
} else if ($this->spouttype == \Box\Spout\Common\Type::CSV) {
    header('Content-Type: text/csv');
}

// Send Content-Disposition header only once
header('Content-Disposition: attachment; filename="' . $filename . '"');
```
**Propósito**: Control total sobre los headers HTTP, enviando cada uno **una sola vez**.

#### 3. Uso de openToFile() en lugar de openToBrowser() (línea 104)
```php
// Open to php://output instead of using openToBrowser
$this->writer->openToFile('php://output');
```
**Propósito**: Escribir directamente a la salida HTTP sin que Spout envíe sus propios headers.

## ¿Por Qué Funciona?

La solución evita completamente el método `openToBrowser()` de Spout, que era el responsable de enviar headers duplicados. En su lugar:

1. Limpiamos todos los buffers de salida
2. Enviamos manualmente los headers necesarios **una sola vez**
3. Usamos `openToFile('php://output')` para escribir directamente al navegador

Esto garantiza que solo se envíe **un único** header `Content-Disposition`, cumpliendo con el estándar HTTP.

## Compatibilidad

✅ **No afecta**:
- Tests automáticos (BEHAT/PHPUNIT)
- Exportación a archivos locales
- Otras funcionalidades de descarga de Moodle
- Plugins de terceros

✅ **Solo afecta**:
- Descargas de reportes de quiz (overview, responses, statistics)
- Cualquier descarga que use dataformat (Excel, CSV, ODS)

## Seguridad

- ✅ Los headers de seguridad se mantienen intactos
- ✅ La validación de permisos no se modifica
- ✅ El contenido del archivo no se ve afectado
- ✅ Compatible con HTTP y HTTPS

## Testing Realizado

### Funciona correctamente:
- ✅ Descarga de reportes de quiz en formato Excel (.xlsx)
- ✅ Descarga de reportes de quiz en formato CSV (.csv)
- ✅ Descarga de reportes de quiz en formato ODS (.ods)
- ✅ Navegadores: Chrome, Firefox, Edge, Safari
- ✅ Conexiones HTTPS

## Reversión (si necesario)

Si necesitas revertir este cambio, simplemente restaura el método `send_http_headers()` original:

```php
public function send_http_headers() {
    $this->writer = \Box\Spout\Writer\Common\Creator\WriterEntityFactory::createWriter($this->spouttype);
    if (method_exists($this->writer, 'setTempFolder')) {
        $this->writer->setTempFolder(make_request_directory());
    }
    $filename = $this->filename . $this->get_extension();
    if (PHPUNIT_TEST) {
        $this->writer->openToFile('php://output');
    } else {
        $this->writer->openToBrowser($filename);
    }
    $this->renamecurrentsheet = true;
}
```

## Información Técnica

- **Versión de Moodle**: Compatible con Moodle 3.x, 4.x
- **Librerías afectadas**: Spout (Box/Spout)
- **Fecha del fix**: Noviembre 2025
- **Reportado por**: Sistema hseqvirtual.com
- **Error original**: ERR_RESPONSE_HEADERS_MULTIPLE_CONTENT_DISPOSITION

## Referencias

- [RFC 6266 - Content-Disposition Header](https://tools.ietf.org/html/rfc6266)
- [MDN - Content-Disposition](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition)
- [Chromium Issue - Multiple Content-Disposition](https://bugs.chromium.org/p/chromium/issues/detail?id=1010232)

## Contacto

Para preguntas o problemas relacionados con este fix, contactar al equipo de desarrollo de HSEQ Virtual.

