Para agregar una funcionalidad que permita a los aprendices tener 3 días para subir su asistencia cuando falten, debes realizar varias modificaciones y adiciones en tu controlador y modelo. Aquí te dejo el código actualizado para que los aprendices puedan subir su asistencia dentro de un plazo de 3 días.

### 1. Modificar el Modelo `ReporteInasistencias`
Agrega una propiedad `FechaLimiteSubida` en el modelo `ReporteInasistencias`:

```csharp
public class ReporteInasistencias
{
    public int Id_reporte { get; set; }
    public string Tipo_reporte { get; set; }
    public int Cantidad_reporte { get; set; }
    public DateTime Fecha_registro { get; set; }
    public int Codigo_asistencia { get; set; }
    public int Id_usuario { get; set; }
    public DateTime FechaLimiteSubida { get; set; } // Nueva propiedad
}
```

### 2. Modificar el Método `ObtenerReporteInasistencias`
Actualiza el método `ObtenerReporteInasistencias` para calcular y almacenar la fecha límite de subida de la asistencia:

```csharp
private List<ReporteInasistencias> ObtenerReporteInasistencias(DateTime fechaInicio, DateTime fechaFin)
{
    List<ReporteInasistencias> reporte = new List<ReporteInasistencias>();

    using (SqlConnection connection = new SqlConnection(Conexion))
    {
        string query = @"
            SELECT id_reporte, tipo_reporte, COUNT(*) AS cantidad_reporte, fecha_registro, codigo_asistencia, id_usuario
            FROM reporte
            WHERE fecha_registro >= @FechaInicio AND fecha_registro <= @FechaFin
            GROUP BY id_reporte, fecha_registro, tipo_reporte, codigo_asistencia, id_usuario";

        SqlCommand command = new SqlCommand(query, connection);
        command.Parameters.AddWithValue("@FechaInicio", fechaInicio);
        command.Parameters.AddWithValue("@FechaFin", fechaFin);

        connection.Open();
        SqlDataReader reader = command.ExecuteReader();
        while (reader.Read())
        {
            ReporteInasistencias item = new ReporteInasistencias
            {
                Id_reporte = (int)reader["id_reporte"],
                Tipo_reporte = reader["tipo_reporte"].ToString(),
                Cantidad_reporte = (int)reader["cantidad_reporte"],
                Fecha_registro = (DateTime)reader["fecha_registro"],
                Codigo_asistencia = (int)reader["codigo_asistencia"],
                Id_usuario = (int)reader["id_usuario"],
                FechaLimiteSubida = ((DateTime)reader["fecha_registro"]).AddDays(3) // Nueva lógica
            };
            reporte.Add(item);
        }
    }

    return reporte;
}
```

### 3. Crear Método para Registrar Asistencia
Crea un nuevo método para registrar la asistencia dentro del plazo de 3 días:

```csharp
public ActionResult RegistrarAsistencia(int idReporte)
{
    using (SqlConnection connection = new SqlConnection(Conexion))
    {
        string query = "SELECT * FROM reporte WHERE id_reporte = @IdReporte";
        SqlCommand command = new SqlCommand(query, connection);
        command.Parameters.AddWithValue("@IdReporte", idReporte);

        connection.Open();
        SqlDataReader reader = command.ExecuteReader();
        if (reader.Read())
        {
            DateTime fechaRegistro = (DateTime)reader["fecha_registro"];
            DateTime fechaLimite = fechaRegistro.AddDays(3);

            if (DateTime.Now <= fechaLimite)
            {
                // Aquí puedes añadir la lógica para que el aprendiz suba la asistencia
                // Por ejemplo, redirigir a un formulario de subida de asistencia
                return RedirectToAction("FormularioSubidaAsistencia", new { idReporte = idReporte });
            }
            else
            {
                return new HttpStatusCodeResult(403, "El plazo para subir la asistencia ha expirado.");
            }
        }
    }

    return new HttpStatusCodeResult(404, "Reporte no encontrado.");
}

public ActionResult FormularioSubidaAsistencia(int idReporte)
{
    // Devuelve la vista donde el aprendiz puede subir la asistencia
    return View(idReporte);
}

[HttpPost]
public ActionResult SubirAsistencia(int idReporte, HttpPostedFileBase file)
{
    if (file != null && file.ContentLength > 0)
    {
        var fileName = Path.GetFileName(file.FileName);
        var path = Path.Combine(Server.MapPath("~/App_Data/uploads"), fileName);
        file.SaveAs(path);

        // Aquí puedes añadir la lógica para guardar la información de la subida de asistencia en la base de datos

        return RedirectToAction("Index");
    }

    return new HttpStatusCodeResult(400, "Archivo no válido.");
}
```

### 4. Crear Vistas para `FormularioSubidaAsistencia`
Crea la vista `FormularioSubidaAsistencia` para permitir a los aprendices subir su asistencia:

#### FormularioSubidaAsistencia.cshtml
```html
@model int

@{
    ViewBag.Title = "Subir Asistencia";
}

<h2>Subir Asistencia</h2>

@using (Html.BeginForm("SubirAsistencia", "Instructor", FormMethod.Post, new { enctype = "multipart/form-data" }))
{
    @Html.HiddenFor(model => model)
    <div class="form-group">
        <label for="file">Seleccionar archivo de asistencia:</label>
        <input type="file" name="file" id="file" class="form-control" />
    </div>
    <button type="submit" class="btn btn-primary">Subir</button>
}
```

### 5. Modificar la Vista de Index para Mostrar Opciones
Actualiza la vista `Index` para mostrar las opciones y acciones para los instructores:

#### Index.cshtml
```html
@{
    ViewBag.Title = "Inicio Instructor";
}

<h2>Reportes de Inasistencias</h2>

<p>Total de inasistencias: @ViewBag.ReporteInasistencias</p>

<h2>Opciones</h2>
<ul>
    <li>@Html.ActionLink("Generar QR", "GenerarQR")</li>
    <li>@Html.ActionLink("Registrar Asistencia", "RegistrarAsistencia")</li>
    <li>@Html.ActionLink("Fichas", "Fichas")</li>
    <li>@Html.ActionLink("Programas de Formación", "ProgramasFormacion")</li>
    <li>@Html.ActionLink("Generar Reporte de Inasistencias", "GenerarReporteInasistencias")</li>
    <li>@Html.ActionLink("Establecer Duración del QR", "EstablecerDuracionQR")</li>
</ul>
```

Con estas modificaciones, los aprendices pueden tener un plazo de 3 días para subir su asistencia en caso de falta. Además, se incluyen las vistas necesarias y se actualiza el controlador para manejar esta funcionalidad.