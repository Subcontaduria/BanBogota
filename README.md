# BanBogota


Documentación de Código: Extractor de PDF del Banco de Bogotá

Resumen del Documento

Este documento describe el Extractor de PDF de BanBogotá, una herramienta web simple que permite a los usuarios extraer datos de transacciones de un estado de cuenta bancario en formato PDF. La aplicación procesa el archivo de forma local en el navegador, garantizando la privacidad de los datos. El código utiliza JavaScript nativo (Vanilla JS) y la biblioteca PDF.js para el procesamiento del PDF y la exportación a CSV.

Estructura y Tecnologías

El código es un único archivo HTML que integra toda la funcionalidad:

    HTML: Define la estructura de la interfaz de usuario, incluyendo un título, un campo de entrada de archivo (<input type="file">), botones de acción y un área para mostrar mensajes de estado. El diseño es minimalista y directo.

    CSS: Los estilos se definen en un bloque <style> y están diseñados para crear una interfaz limpia y funcional. Se usan selectores simples para estilizar los botones y el contenedor principal, proporcionando una experiencia de usuario clara.

    JavaScript: La lógica principal reside en un bloque <script> al final del cuerpo del documento.

        PDF.js: La biblioteca se carga desde un CDN para manejar el análisis del archivo PDF y la extracción de texto.

        Manejo de Archivos: La aplicación utiliza la API FileReader para leer el archivo PDF como un ArrayBuffer.

        Expresiones Regulares (Regex): Se utiliza una expresión regular compleja para identificar y capturar patrones de transacciones que incluyen fechas, descripciones, movimientos y saldos. Esta es la parte central de la lógica de extracción.

        Manipulación del DOM: El código interactúa directamente con los elementos del documento para manejar eventos y actualizar el mensaje de estado para el usuario.

Flujo de la Aplicación

El proceso de extracción de datos del PDF sigue un flujo de trabajo simple y lineal:

    Carga del Archivo: El usuario selecciona un archivo PDF a través del campo de entrada. El nombre del archivo se almacena y se prepara para el procesamiento.

    Procesamiento del PDF: Al hacer clic en el botón "Procesar PDF", la función asíncrona procesarPDF() se activa.

        La función lee el archivo PDF en la memoria del navegador utilizando FileReader.

        PDF.js carga el documento, y luego la aplicación itera sobre cada página para extraer el texto completo.

        Se aplica una expresión regular al texto extraído de cada página para encontrar todas las transacciones que coinciden con el patrón esperado.

        Por cada coincidencia, los datos (fecha, detalle, movimiento, saldo) se extraen y se almacenan en un arreglo global llamado transacciones.

        Una función auxiliar, normalizarNumero(), limpia los valores numéricos, reemplazando los separadores de miles (.) y decimales (,) para asegurar un formato consistente.

        Al finalizar el proceso, se muestra un mensaje de éxito en la interfaz, indicando cuántas transacciones se detectaron.

    Exportación a CSV: Al hacer clic en "Exportar a CSV", la función exportarCSV() se ejecuta.

        Esta función genera un archivo CSV a partir del arreglo transacciones.

        Se crea una fila de encabezados (Fecha|Detalle|Movimiento|Saldo).

        Cada transacción se mapea a una fila en formato de texto, uniendo los campos con el separador |.

        Finalmente, la aplicación crea un objeto Blob con el contenido del CSV y activa su descarga en el navegador.

Comentarios de Código (Inline)

<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Extractor PDF BanBogotá</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js"></script>
  <style>
    /* Estilos para el cuerpo de la página y el contenedor principal. */
    body {
      font-family: Arial, sans-serif;
      background: #f8f9fa;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      margin: 0;
    }
    .contenedor {
      background: #fff;
      padding: 30px;
      border-radius: 10px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.1);
      text-align: center;
      width: 400px;
    }
    h1 {
      color: #007bff;
      font-size: 22px;
      margin-bottom: 20px;
    }
    input[type="file"] {
      margin: 15px 0;
    }
    button {
      padding: 10px 18px;
      margin: 8px;
      border: none;
      border-radius: 5px;
      font-size: 14px;
      cursor: pointer;
      transition: 0.2s;
    }
    /* Estilos específicos para los botones de acción. */
    button:first-of-type {
      background: #6c757d;
      color: white;
    }
    button:first-of-type:hover {
      background: #5a6268;
    }
    button:last-of-type {
      background: #007bff;
      color: white;
    }
    button:last-of-type:hover {
      background: #0056b3;
    }
    #mensaje {
      margin-top: 15px;
      font-size: 14px;
      color: #555;
    }
  </style>
</head>
<body>
  <div class="contenedor">
    <h1>Extractor PDF BanBogotá</h1>
    <input type="file" id="fileInput" accept="application/pdf"><br>
    <button onclick="procesarPDF()">Procesar PDF</button>
    <button onclick="exportarCSV()">Exportar a CSV</button>
    <div id="mensaje">Seleccione un archivo PDF para comenzar.</div>
  </div>

  <script>
    // Variables globales para almacenar los datos extraídos y el nombre del archivo.
    let transacciones = [];
    let nombreArchivo = "export.csv";

    /**
     * Normaliza un valor numérico para exportarlo a CSV.
     * Reemplaza el punto decimal por una coma y elimina los separadores de miles.
     * @param {string} valor El valor numérico como cadena de texto.
     * @returns {string} El valor normalizado.
     */
    function normalizarNumero(valor) {
      if (!valor) return "";
      let limpio = valor.replace(/,/g, "");
      limpio = limpio.replace(".", ",");
      return limpio;
    }

    /**
     * Función principal asíncrona que lee el archivo PDF, extrae el texto y busca transacciones.
     */
    async function procesarPDF() {
      const file = document.getElementById('fileInput').files[0];
      if (!file) {
        alert("Por favor selecciona un PDF.");
        return;
      }
      // Almacena el nombre del archivo con extensión CSV.
      nombreArchivo = file.name.replace(/\.pdf$/i, ".csv");

      const fileReader = new FileReader();
      fileReader.onload = async function() {
        const typedarray = new Uint8Array(this.result);
        const pdf = await pdfjsLib.getDocument(typedarray).promise;

        transacciones = [];
        // Itera sobre cada página para extraer el texto.
        for (let i = 1; i <= pdf.numPages; i++) {
          const page = await pdf.getPage(i);
          const textContent = await page.getTextContent();
          const texto = textContent.items.map(t => t.str).join(" ");

          // Expresión regular para el formato de transacción de BanBogotá:
          // (\d{2}\/\d{2}): Captura la fecha (DD/MM).
          // ([\s\S]+?): Captura el detalle de la transacción, de forma no codiciosa.
          // (-?\d{1,3}(?:,\d{3})*\.\d{2}): Captura el movimiento (número con punto decimal, opcionalmente negativo).
          // (-?\d{1,3}(?:,\d{3})*\.\d{2}): Captura el saldo (número con punto decimal, opcionalmente negativo).
          const regex = /(\d{2}\/\d{2})\s+([\s\S]+?)\s+(-?\d{1,3}(?:,\d{3})*\.\d{2})\s+(-?\d{1,3}(?:,\d{3})*\.\d{2})/g;
          let match;
          // Itera a través de todas las coincidencias en el texto de la página.
          while ((match = regex.exec(texto)) !== null) {
            transacciones.push({
              fecha: match[1],
              detalle: match[2].trim(),
              movimiento: normalizarNumero(match[3]),
              saldo: normalizarNumero(match[4])
            });
          }
        }

        document.getElementById('mensaje').innerHTML = 
          `✔ Procesamiento completo: ${transacciones.length} transacciones detectadas`;
      };
      // Lee el contenido del archivo como un ArrayBuffer.
      fileReader.readAsArrayBuffer(file);
    }

    /**
     * Exporta las transacciones extraídas a un archivo CSV.
     */
    function exportarCSV() {
      if (transacciones.length === 0) {
        alert("No hay transacciones para exportar.");
        return;
      }
      const encabezados = ["Fecha", "Detalle", "Movimiento", "Saldo"];
      // Crea las filas del CSV uniendo los valores con '|'.
      const filas = transacciones.map(t => 
        [t.fecha, t.detalle, t.movimiento, t.saldo].join("|")
      );
      // Combina los encabezados y las filas en un solo string.
      const csvContent = [encabezados.join("|"), ...filas].join("\n");
      const blob = new Blob([csvContent], { type: "text/csv;charset=utf-8;" });
      const link = document.createElement("a");
      link.href = URL.createObjectURL(blob);
      link.download = nombreArchivo;
      link.click();
    }
  </script>
</body>
</html>
  
