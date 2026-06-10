# specs.md v2 — Spec-Driven Development (SDD)
## Sistema de Gestión de Consorcios y Amenities
**Stack:** .NET 10 · EF Core 10 · PostgreSQL · Npgsql · Clean N-Layer (ProyectoBase)
**Table prefix:** `PB_` · **DI:** Scoped · **Response envelope:** `ServiceResponse<T>` / `PagedResponse<T>`
**Tests:** xUnit + Moq + FluentAssertions

> **Instrucción para el agente:** buscar el `SPEC-ID` del feature solicitado, leer sus secciones en orden, implementar sin interpretación adicional. Cada campo, constraint y nombre de archivo es definitivo.

---

## ÍNDICE

| SPEC-ID | Feature | Estado | Depende de | Migración |
|---|---|---|---|---|
| [SPEC-ARCH](#spec-arch) | Arquitectura, convenciones y trinomio genérico | ✅ | — | 001 |
| [SPEC-ENT](#spec-ent) | Entidades de dominio + Fluent API completa | ✅ CU08 / 🔲 resto | SPEC-ARCH | 002 |
| [SPEC-CU08](#spec-cu08) | Onboarding Consorcio/Complejo/Amenity | ✅ | SPEC-ENT | 002 |
| [SPEC-CU01](#spec-cu01) | Reservar Amenity | 🔲 | SPEC-ENT, SPEC-CU08 | 003 |
| [SPEC-CU02](#spec-cu02) | Reportar Incidencia | 🔲 | SPEC-ENT | 003 |
| [SPEC-CU04](#spec-cu04) | Resolver Incidencia / Rehabilitar Amenity | 🔲 | SPEC-CU02 | — |
| [SPEC-CU03](#spec-cu03) | Control de Acceso de Invitados | 🔲 | SPEC-ENT | 004 |
| [SPEC-CU05](#spec-cu05) | Lista de Espera | 🔲 | SPEC-CU01 | 005 |
| [SPEC-CU06](#spec-cu06) | Sancionar Unidad Habitacional | 🔲 | SPEC-ENT | — |
| [SPEC-CU07](#spec-cu07) | Pago de Reserva | 🔲 | SPEC-CU01 | — |
| [SPEC-CU09](#spec-cu09) | Reportes y Auditoría | 🔲 | SPEC-ENT | 006 |
| [SPEC-CU10](#spec-cu10) | Mantenimiento Programado | 🔲 | SPEC-ENT | 007 |
| [SPEC-CU11](#spec-cu11) | Baja Inquilino / Cambio Ocupante | 🔲 | SPEC-ENT | — |
| [SPEC-NOTIF](#spec-notif) | Notificaciones (Port & Adapter) | 🔲 | SPEC-CU01, SPEC-CU02 | — |
| [SPEC-AUTH](#spec-auth) | Autenticación, Roles, JWT | 🔲 | SPEC-ENT | 008 |

---

---

## SPEC-ARCH
### Arquitectura Base y Convenciones Globales

#### Capas y flujo de dependencias
```
[Cliente HTTP]
  → GlobalErrorHandlingMiddleware
  → Controller : GenericControllerAsync<T>   (o controller propio para orquestadores)
  → IServiceAsync<T> : ServiceAsync<T>       (o IXxxService propio)
  → IRepositoryAsync<T> : RepositoryAsync<T>
  → ApplicationDbContext → PostgreSQL
```

#### Reglas de oro — el agente NO debe violarlas

| # | Regla |
|---|---|
| 1 | Toda tabla: `entity.ToTable("PB_NombreEntidad")` en Fluent API. Clase C# sin prefijo. |
| 2 | Cero `DataAnnotations` en entidades. Todas las constraints en `OnModelCreating`. |
| 3 | Todo servicio, repo y adapter: `builder.Services.AddScoped<Interface, Impl>()`. |
| 4 | Todo endpoint retorna `Ok(new ServiceResponse<T>(data))` o lanza excepción de dominio. |
| 5 | Error de negocio: `throw new BadRequestException("[BR-ID] Mensaje.")` desde el servicio. |
| 6 | No encontrado: `throw new NotFoundException("Mensaje.")` desde el servicio. |
| 7 | El middleware convierte `BadRequestException→400`, `NotFoundException→404`, resto→500. |
| 8 | El registro genérico abierto ya resuelve CRUD simple: `services.AddScoped(typeof(IServiceAsync<>), typeof(ServiceAsync<>))`. Sólo registrar explícitamente servicios específicos/orquestadores. |
| 9 | Servicios con lógica extra heredan `ServiceAsync<T>` y sobrescriben `Create`/`Update`. |
| 10 | Casos de uso multi-entidad: servicio orquestador propio (NO hereda `ServiceAsync<T>`), con `IDbContextTransaction` explícita. |

#### Convención de nombrado de BRs

```
BR-{MÓDULO}-{NNN}   →   [BR-ONB-001], [BR-RES-001], etc.
Formato en errorMessage: "[BR-ID] Oración descriptiva en español."
```

| Prefijo | Módulo |
|---|---|
| `BR-ONB-` | Onboarding (CU-08) |
| `BR-RES-` | Reservas (CU-01) |
| `BR-INC-` | Incidencias (CU-02 / CU-04) |
| `BR-ACC-` | Acceso de Invitados (CU-03) |
| `BR-ESP-` | Lista de Espera (CU-05) |
| `BR-SAN-` | Sanciones (CU-06) |
| `BR-PAG-` | Pagos (CU-07) |
| `BR-AUD-` | Auditoría / Reportes (CU-09) |
| `BR-MAN-` | Mantenimiento Programado (CU-10) |
| `BR-BAJ-` | Baja Inquilino (CU-11) |
| `BR-NOT-` | Notificaciones |
| `BR-AUTH-` | Autenticación y Roles |

#### Formato de respuesta estándar
```json
{ "data": {}, "success": true,  "errorMessage": null }
{ "data": null, "success": false, "errorMessage": "[BR-ID] Mensaje." }
{ "data": { "data":[], "page":1, "limit":10, "totalRows":N, "totalPage":M }, "success": true, "errorMessage": null }
```

#### Convención de migrations

```powershell
dotnet ef migrations add {NombreMigration} --output-dir Migrations
dotnet ef migrations script --output "scripts/{NNN}_{NombreMigration}.sql" --idempotent
dotnet ef database update
```
Numeración: `001_InitialCreate`, `002_CU08_OnboardingDomain`, `003_CU01_CU02_ReservaIncidencia`, …

#### Pattern Port & Adapter (para todos los servicios externos)
```
Core define interface:        Services/Ports/IXxxPort.cs
Infra implementa:             Infrastructure/Adapters/XxxAdapter.cs
Registro en Program.cs:       builder.Services.AddScoped<IXxxPort, XxxAdapter>();
```

---

---

## SPEC-ENT
### Entidades de Dominio — POCOs + Fluent API Completa

> El agente debe agregar cada bloque de `DbSet` y `OnModelCreating` en `ApplicationDbContext.cs` al momento de implementar el SPEC-ID correspondiente.

---

### ENT-01 · Consorcio · (CU-08)
```csharp
// Models/Consorcio.cs
public class Consorcio {
    public int    IDConsorcio { get; set; }
    public string Cuit        { get; set; }   // 11 dígitos, único global
    public string Nombre      { get; set; }
    public string Email       { get; set; }   // único global
    public string Telefono    { get; set; }   // nullable
}
```
```csharp
// ApplicationDbContext — Fluent API
modelBuilder.Entity<Consorcio>(e => {
    e.ToTable("PB_Consorcio");
    e.HasKey(x => x.IDConsorcio);
    e.Property(x => x.Cuit).IsRequired().HasMaxLength(11);
    e.HasIndex(x => x.Cuit).IsUnique();
    e.Property(x => x.Nombre).IsRequired().HasMaxLength(100);
    e.Property(x => x.Email).IsRequired().HasMaxLength(100);
    e.HasIndex(x => x.Email).IsUnique();
    e.Property(x => x.Telefono).HasMaxLength(20);
});
```

---

### ENT-02 · Complejo · (CU-08)
```csharp
// Models/Complejo.cs
public class Complejo {
    public int    IDComplejo  { get; set; }
    public int    IDConsorcio { get; set; }   // FK → Consorcio
    public string Nombre      { get; set; }
    public string Tipo        { get; set; }   // "EDIFICIO" | "BARRIO_PRIVADO"
    public string Direccion   { get; set; }
    public Consorcio Consorcio { get; set; }
}
```
```csharp
modelBuilder.Entity<Complejo>(e => {
    e.ToTable("PB_Complejo");
    e.HasKey(x => x.IDComplejo);
    e.Property(x => x.Nombre).IsRequired().HasMaxLength(100);
    e.HasIndex(x => new { x.IDConsorcio, x.Nombre }).IsUnique();
    e.Property(x => x.Tipo).IsRequired().HasMaxLength(20);
    e.Property(x => x.Direccion).IsRequired().HasMaxLength(200);
    e.HasOne(x => x.Consorcio).WithMany()
     .HasForeignKey(x => x.IDConsorcio).OnDelete(DeleteBehavior.Restrict);
});
```

---

### ENT-03 · UnidadHabitacional · (CU-08)
```csharp
// Models/UnidadHabitacional.cs
public class UnidadHabitacional {
    public int     IDUnidadHabitacional { get; set; }
    public int     IDComplejo           { get; set; }   // FK → Complejo
    public string  Identificador        { get; set; }   // "1A", "Lote 42"
    public bool    DebeExpensas         { get; set; }   // default: false
    public decimal SaldoActual          { get; set; }   // default: 0.00
    public string  EstadoUnidad         { get; set; }   // "ACTIVA" | "SUSPENDIDA"  default: "ACTIVA"
    public int     ContadorInfracciones { get; set; }   // default: 0
    public Complejo Complejo            { get; set; }
}
```
```csharp
modelBuilder.Entity<UnidadHabitacional>(e => {
    e.ToTable("PB_UnidadHabitacional");
    e.HasKey(x => x.IDUnidadHabitacional);
    e.Property(x => x.Identificador).IsRequired().HasMaxLength(20);
    e.HasIndex(x => new { x.IDComplejo, x.Identificador }).IsUnique();
    e.Property(x => x.DebeExpensas).HasDefaultValue(false);
    e.Property(x => x.SaldoActual).HasColumnType("decimal(12,2)").HasDefaultValue(0.00m);
    e.Property(x => x.EstadoUnidad).IsRequired().HasMaxLength(15).HasDefaultValue("ACTIVA");
    e.Property(x => x.ContadorInfracciones).HasDefaultValue(0);
    e.HasOne(x => x.Complejo).WithMany()
     .HasForeignKey(x => x.IDComplejo).OnDelete(DeleteBehavior.Restrict);
});
```

---

### ENT-04 · Amenity · (CU-08)
```csharp
// Models/Amenity.cs
public class Amenity {
    public int    IDAmenity  { get; set; }
    public int    IDComplejo { get; set; }   // FK → Complejo
    public string Nombre     { get; set; }
    public int    Capacidad  { get; set; }   // min: 1
    public string Estado     { get; set; }   // "DISPONIBLE" | "FUERA_DE_SERVICIO"  default: "DISPONIBLE"
    public Complejo      Complejo { get; set; }
    public AmenityConfig Config   { get; set; }
}
```
```csharp
modelBuilder.Entity<Amenity>(e => {
    e.ToTable("PB_Amenity");
    e.HasKey(x => x.IDAmenity);
    e.Property(x => x.Nombre).IsRequired().HasMaxLength(50);
    e.HasIndex(x => new { x.IDComplejo, x.Nombre }).IsUnique();
    e.Property(x => x.Capacidad).IsRequired();
    e.Property(x => x.Estado).IsRequired().HasMaxLength(20).HasDefaultValue("DISPONIBLE");
    e.HasOne(x => x.Complejo).WithMany()
     .HasForeignKey(x => x.IDComplejo).OnDelete(DeleteBehavior.Restrict);
});
```

---

### ENT-05 · AmenityConfig · (CU-08 — RN-34)
```csharp
// Models/AmenityConfig.cs
public class AmenityConfig {
    public int      IDAmenityConfig         { get; set; }
    public int      IDAmenity               { get; set; }   // FK → Amenity (1:1)
    public TimeOnly HorarioInicio           { get; set; }
    public TimeOnly HorarioFin              { get; set; }   // constraint: > HorarioInicio
    public int      DuracionBloqueMinutos   { get; set; }   // min: 1
    public int      TiempoLimpiezaMinutos   { get; set; }   // min: 0; < DuracionBloqueMinutos; default: 0
    public decimal  Tarifa                  { get; set; }   // min: 0; default: 0.00
    public int      LimiteReservasMesUnidad { get; set; }   // min: 1
    public bool     RequiereAprobacion      { get; set; }   // default: false
    public Amenity  Amenity                 { get; set; }
}
```
```csharp
modelBuilder.Entity<AmenityConfig>(e => {
    e.ToTable("PB_AmenityConfig");
    e.HasKey(x => x.IDAmenityConfig);
    e.HasIndex(x => x.IDAmenity).IsUnique();
    e.Property(x => x.HorarioInicio).IsRequired().HasColumnType("time");
    e.Property(x => x.HorarioFin).IsRequired().HasColumnType("time");
    e.Property(x => x.DuracionBloqueMinutos).IsRequired();
    e.Property(x => x.TiempoLimpiezaMinutos).HasDefaultValue(0);
    e.Property(x => x.Tarifa).HasColumnType("decimal(10,2)").HasDefaultValue(0.00m);
    e.Property(x => x.LimiteReservasMesUnidad).IsRequired();
    e.Property(x => x.RequiereAprobacion).HasDefaultValue(false);
    e.HasOne(x => x.Amenity).WithOne(x => x.Config)
     .HasForeignKey<AmenityConfig>(x => x.IDAmenity).OnDelete(DeleteBehavior.Cascade);
});
```

---

### ENT-06 · Inquilino · (CU-11)
```csharp
// Models/Inquilino.cs
public class Inquilino {
    public int    IDInquilino          { get; set; }
    public int    IDUnidadHabitacional { get; set; }   // FK → UnidadHabitacional
    public string Nombre               { get; set; }
    public string Apellido             { get; set; }
    public string Dni                  { get; set; }
    public string Telefono             { get; set; }   // nullable
    public bool   Activo               { get; set; }   // default: true; false = dado de baja
    public UnidadHabitacional UnidadHabitacional { get; set; }
}
```
```csharp
modelBuilder.Entity<Inquilino>(e => {
    e.ToTable("PB_Inquilino");
    e.HasKey(x => x.IDInquilino);
    e.Property(x => x.Nombre).IsRequired().HasMaxLength(100);
    e.Property(x => x.Apellido).IsRequired().HasMaxLength(100);
    e.Property(x => x.Dni).IsRequired().HasMaxLength(20);
    e.HasIndex(x => x.Dni).IsUnique();
    e.Property(x => x.Telefono).HasMaxLength(20);
    e.Property(x => x.Activo).HasDefaultValue(true);
    e.HasOne(x => x.UnidadHabitacional).WithMany()
     .HasForeignKey(x => x.IDUnidadHabitacional).OnDelete(DeleteBehavior.Restrict);
});
```

---

### ENT-07 · Invitado · (CU-03)
```csharp
// Models/Invitado.cs
public class Invitado {
    public int      IDInvitado           { get; set; }
    public int      IDUnidadHabitacional { get; set; }   // FK → UnidadHabitacional
    public string   Nombre               { get; set; }
    public string   Apellido             { get; set; }
    public string   Dni                  { get; set; }
    public string   EstadoAcceso         { get; set; }   // "PERMITIDO" | "DENEGADO"  default: "PERMITIDO"
    public DateTime? HoraIngreso         { get; set; }   // nullable; se completa al ingresar
    public DateTime? HoraEgreso          { get; set; }   // nullable; se completa al egresar (RN-29)
    public UnidadHabitacional UnidadHabitacional { get; set; }
}
```
```csharp
modelBuilder.Entity<Invitado>(e => {
    e.ToTable("PB_Invitado");
    e.HasKey(x => x.IDInvitado);
    e.Property(x => x.Nombre).IsRequired().HasMaxLength(100);
    e.Property(x => x.Apellido).IsRequired().HasMaxLength(100);
    e.Property(x => x.Dni).IsRequired().HasMaxLength(20);
    e.Property(x => x.EstadoAcceso).IsRequired().HasMaxLength(15).HasDefaultValue("PERMITIDO");
    e.Property(x => x.HoraIngreso).HasColumnType("timestamptz");
    e.Property(x => x.HoraEgreso).HasColumnType("timestamptz");
    e.HasOne(x => x.UnidadHabitacional).WithMany()
     .HasForeignKey(x => x.IDUnidadHabitacional).OnDelete(DeleteBehavior.Restrict);
});
```

---

### ENT-08 · Reserva · (CU-01)
```csharp
// Models/Reserva.cs
public class Reserva {
    public int      IDReserva            { get; set; }
    public int      IDAmenity            { get; set; }   // FK → Amenity
    public int      IDUnidadHabitacional { get; set; }   // FK → UnidadHabitacional
    public DateOnly FechaUso             { get; set; }
    public TimeOnly HoraInicio           { get; set; }
    public TimeOnly HoraFin              { get; set; }   // calculado: HoraInicio + DuracionBloqueMinutos
    public int      CantidadInvitados    { get; set; }   // default: 0
    public string   Estado               { get; set; }
    // Estados: "PENDIENTE_PAGO" | "PENDIENTE_APROBACION" | "CONFIRMADA" | "CANCELADA" | "EN_ESPERA"
    public DateTime FechaSolicitud       { get; set; }   // UTC; set en el servicio
    public Amenity            Amenity            { get; set; }
    public UnidadHabitacional UnidadHabitacional { get; set; }
}
```
```csharp
modelBuilder.Entity<Reserva>(e => {
    e.ToTable("PB_Reserva");
    e.HasKey(x => x.IDReserva);
    e.Property(x => x.FechaUso).IsRequired().HasColumnType("date");
    e.Property(x => x.HoraInicio).IsRequired().HasColumnType("time");
    e.Property(x => x.HoraFin).IsRequired().HasColumnType("time");
    e.Property(x => x.CantidadInvitados).HasDefaultValue(0);
    e.Property(x => x.Estado).IsRequired().HasMaxLength(25);
    e.Property(x => x.FechaSolicitud).IsRequired().HasColumnType("timestamptz");
    e.HasOne(x => x.Amenity).WithMany()
     .HasForeignKey(x => x.IDAmenity).OnDelete(DeleteBehavior.Restrict);
    e.HasOne(x => x.UnidadHabitacional).WithMany()
     .HasForeignKey(x => x.IDUnidadHabitacional).OnDelete(DeleteBehavior.Restrict);
});
```

---

### ENT-09 · Incidencia · (CU-02)
```csharp
// Models/Incidencia.cs
public class Incidencia {
    public int      IDIncidencia         { get; set; }
    public int      IDAmenity            { get; set; }   // FK → Amenity
    public int      IDUnidadHabitacional { get; set; }   // FK → UnidadHabitacional (quién reporta)
    public string   Descripcion          { get; set; }
    public string   Estado               { get; set; }   // "ABIERTA" | "EN_REPARACION" | "RESUELTA"
    public string   DetalleResolucion    { get; set; }   // nullable; se completa al resolver (CU-04)
    public decimal? CostoEstimado        { get; set; }   // nullable; se completa al resolver (CU-04)
    public DateTime FechaReporte         { get; set; }   // UTC
    public DateTime? FechaResolucion     { get; set; }   // nullable; UTC; set al resolver
    public Amenity            Amenity            { get; set; }
    public UnidadHabitacional UnidadHabitacional { get; set; }
}
```
```csharp
modelBuilder.Entity<Incidencia>(e => {
    e.ToTable("PB_Incidencia");
    e.HasKey(x => x.IDIncidencia);
    e.Property(x => x.Descripcion).IsRequired().HasMaxLength(500);
    e.Property(x => x.Estado).IsRequired().HasMaxLength(20);
    e.Property(x => x.DetalleResolucion).HasMaxLength(500);
    e.Property(x => x.CostoEstimado).HasColumnType("decimal(10,2)");
    e.Property(x => x.FechaReporte).IsRequired().HasColumnType("timestamptz");
    e.Property(x => x.FechaResolucion).HasColumnType("timestamptz");
    e.HasOne(x => x.Amenity).WithMany()
     .HasForeignKey(x => x.IDAmenity).OnDelete(DeleteBehavior.Restrict);
    e.HasOne(x => x.UnidadHabitacional).WithMany()
     .HasForeignKey(x => x.IDUnidadHabitacional).OnDelete(DeleteBehavior.Restrict);
});
```

---

### ENT-10 · ListaEspera · (CU-05)
```csharp
// Models/ListaEspera.cs
public class ListaEspera {
    public int      IDListaEspera        { get; set; }
    public int      IDAmenity            { get; set; }   // FK → Amenity
    public int      IDUnidadHabitacional { get; set; }   // FK → UnidadHabitacional
    public DateOnly FechaUso             { get; set; }
    public TimeOnly HoraInicio           { get; set; }
    public int      Posicion             { get; set; }   // orden FIFO
    public DateTime FechaInscripcion     { get; set; }   // UTC
    public string   Estado               { get; set; }   // "ESPERANDO" | "NOTIFICADO" | "EXPIRADO" | "CONFIRMADO"
    public Amenity            Amenity            { get; set; }
    public UnidadHabitacional UnidadHabitacional { get; set; }
}
```
```csharp
modelBuilder.Entity<ListaEspera>(e => {
    e.ToTable("PB_ListaEspera");
    e.HasKey(x => x.IDListaEspera);
    e.Property(x => x.FechaUso).IsRequired().HasColumnType("date");
    e.Property(x => x.HoraInicio).IsRequired().HasColumnType("time");
    e.Property(x => x.Posicion).IsRequired();
    e.Property(x => x.FechaInscripcion).IsRequired().HasColumnType("timestamptz");
    e.Property(x => x.Estado).IsRequired().HasMaxLength(15).HasDefaultValue("ESPERANDO");
    e.HasOne(x => x.Amenity).WithMany()
     .HasForeignKey(x => x.IDAmenity).OnDelete(DeleteBehavior.Restrict);
    e.HasOne(x => x.UnidadHabitacional).WithMany()
     .HasForeignKey(x => x.IDUnidadHabitacional).OnDelete(DeleteBehavior.Restrict);
});
```

---

### ENT-11 · MantenimientoProgramado · (CU-10)
```csharp
// Models/MantenimientoProgramado.cs
public class MantenimientoProgramado {
    public int      IDMantenimiento { get; set; }
    public int      IDAmenity       { get; set; }   // FK → Amenity
    public string   Descripcion     { get; set; }
    public string   Recurrencia     { get; set; }   // "LUNES".."DOMINGO" | "DIARIO" | "SEMANAL"
    public TimeOnly HoraInicio      { get; set; }
    public TimeOnly HoraFin         { get; set; }   // > HoraInicio
    public DateOnly FechaInicio     { get; set; }
    public DateOnly FechaFin        { get; set; }   // > FechaInicio
    public Amenity  Amenity         { get; set; }
}
```
```csharp
modelBuilder.Entity<MantenimientoProgramado>(e => {
    e.ToTable("PB_MantenimientoProgramado");
    e.HasKey(x => x.IDMantenimiento);
    e.Property(x => x.Descripcion).IsRequired().HasMaxLength(200);
    e.Property(x => x.Recurrencia).IsRequired().HasMaxLength(20);
    e.Property(x => x.HoraInicio).IsRequired().HasColumnType("time");
    e.Property(x => x.HoraFin).IsRequired().HasColumnType("time");
    e.Property(x => x.FechaInicio).IsRequired().HasColumnType("date");
    e.Property(x => x.FechaFin).IsRequired().HasColumnType("date");
    e.HasOne(x => x.Amenity).WithMany()
     .HasForeignKey(x => x.IDAmenity).OnDelete(DeleteBehavior.Restrict);
});
```

---

### ENT-12 · AuditLog · (CU-09 — RN-38)
```csharp
// Models/AuditLog.cs
public class AuditLog {
    public int      IDAuditLog  { get; set; }
    public string   Usuario     { get; set; }   // email o identificador del actor
    public string   Accion      { get; set; }   // "CREAR_RESERVA" | "CANCELAR_RESERVA" | "REGISTRAR_INCIDENCIA" | "SANCIONAR_UNIDAD" | "MODIFICAR_CONFIG"
    public string   Entidad     { get; set; }   // nombre de clase C#: "Reserva", "Incidencia", etc.
    public int      EntidadId   { get; set; }
    public DateTime FechaHora   { get; set; }   // UTC
    public string   Detalle     { get; set; }   // JSON serializado del payload de la acción
}
```
```csharp
modelBuilder.Entity<AuditLog>(e => {
    e.ToTable("PB_AuditLog");
    e.HasKey(x => x.IDAuditLog);
    e.Property(x => x.Usuario).IsRequired().HasMaxLength(100);
    e.Property(x => x.Accion).IsRequired().HasMaxLength(50);
    e.Property(x => x.Entidad).IsRequired().HasMaxLength(50);
    e.Property(x => x.EntidadId).IsRequired();
    e.Property(x => x.FechaHora).IsRequired().HasColumnType("timestamptz");
    e.Property(x => x.Detalle).HasColumnType("text");
});
```

---

---

## SPEC-CU08
### CU-08: Onboarding Consorcio / Complejo / Amenity

**Tipo:** Orquestador multi-entidad (NO hereda `ServiceAsync<T>`)
**Estado:** ✅ Implementado
**Migración:** `002_CU08_OnboardingDomain`
**Entidades:** ENT-01, ENT-02, ENT-03, ENT-04, ENT-05

#### Archivos

```
Models/               Consorcio.cs, Complejo.cs, UnidadHabitacional.cs, Amenity.cs, AmenityConfig.cs
DTOs/CU08/            ConsorcioDto.cs, ComplejoDto.cs, UnidadDto.cs
                      AmenityConfigDto.cs, AmenityCreacionDto.cs
                      OnboardingRequestDto.cs, OnboardingResponseDto.cs
Services/OnboardingService/   IOnboardingService.cs, OnboardingService.cs
Services/Ports/       IResidentNotificationPort.cs
Infrastructure/Adapters/      EmailNotificationAdapter.cs
Controllers/          OnboardingController.cs
```

#### DTOs

```csharp
// Input
public class ConsorcioDto     { public string Cuit, Nombre, Email, Telefono { get; set; } }
public class ComplejoDto      { public string Nombre, Tipo, Direccion { get; set; } }
public class UnidadDto        { public string Identificador, EmailResidente { get; set; } }
public class AmenityConfigDto {
    public string  HorarioInicio, HorarioFin { get; set; }        // "HH:mm"
    public int     DuracionBloqueMinutos, TiempoLimpiezaMinutos, LimiteReservasMesUnidad { get; set; }
    public decimal Tarifa { get; set; }
    public bool    RequiereAprobacion { get; set; }
}
public class AmenityCreacionDto { public string Nombre { get; set; } public int Capacidad { get; set; } public AmenityConfigDto Config { get; set; } }
public class OnboardingRequestDto {
    public ConsorcioDto Consorcio { get; set; }           // Required
    public ComplejoDto  Complejo  { get; set; }           // Required
    public List<UnidadDto>          Unidades  { get; set; }  // Required, min: 1
    public List<AmenityCreacionDto> Amenities { get; set; }  // Optional
}

// Output
public class OnboardingResponseDto {
    public int       IDConsorcio, IDComplejo { get; set; }
    public List<int> UnidadesIds, AmenitiesIds { get; set; }
    public string    Status { get; set; }   // "ONBOARDING_COMPLETE"
}
```

#### Business Rules — evaluación pre-persistencia

| BR-ID | P | Condición | Acción |
|---|---|---|---|
| `BR-ONB-001` | 1 | `Cuit` existe en `PB_Consorcio` | `BadRequestException("[BR-ONB-001] Ya existe un consorcio con el CUIT {cuit}.")` |
| `BR-ONB-003` | 1 | `Tipo` ∉ `{"EDIFICIO","BARRIO_PRIVADO"}` | `BadRequestException("[BR-ONB-003] Tipo de complejo inválido. Valores: EDIFICIO, BARRIO_PRIVADO.")` |
| `BR-RN34-001` | 2 | `HorarioFin <= HorarioInicio` | `BadRequestException("[BR-RN34-001] HorarioFin debe ser posterior a HorarioInicio.")` |
| `BR-RN34-002` | 2 | `DuracionBloqueMinutos<=0` OR `Capacidad<=0` OR `LimiteReservasMesUnidad<=0` | `BadRequestException("[BR-RN34-002] DuracionBloque, Capacidad y LimiteReservas deben ser enteros positivos.")` |
| `BR-RN34-003` | 2 | `Tarifa < 0` | `BadRequestException("[BR-RN34-003] La tarifa no puede ser negativa.")` |
| `BR-RN34-004` | 2 | `TiempoLimpiezaMinutos >= DuracionBloqueMinutos` | `BadRequestException("[BR-RN34-004] TiempoLimpieza debe ser menor que DuracionBloque.")` |
| `BR-ONB-002` | 3 | `Nombre` amenity existe en mismo `IDComplejo` | `BadRequestException("[BR-ONB-002] Ya existe un amenity '{nombre}' en este complejo.")` |
| `BR-DEFAULT` | — | `Estado == null` | silencioso → `"DISPONIBLE"` |

**Nota BR-ONB-002:** evaluar DENTRO de la transacción, luego de crear el Complejo (requiere IDComplejo).

#### Flujo de implementación — OnboardingService.ExecuteAsync

```
PASO 0  null-check request, Consorcio, Complejo, Unidades; Unidades.Count >= 1
PASO 1  Evaluar P1: BR-ONB-001, BR-ONB-003
        Evaluar P2: BR-RN34-001..004 por cada amenity del payload
PASO 2  BEGIN TRANSACTION (IDbContextTransaction)
  S1    Insert Consorcio
  S2    Insert Complejo (IDConsorcio = consorcio.IDConsorcio)
  S3    Task.WhenAll → Insert todas las Unidades (IDComplejo = complejo.IDComplejo)
  S4    foreach amenityDto:
          Evaluar BR-ONB-002 (ya tenemos IDComplejo)
          Insert Amenity
          Insert AmenityConfig (IDAmenity = amenity.IDAmenity)
  S5    COMMIT
PASO 3  EnqueueResidentInvitationsAsync (fuera de tx — Outbox)
        Si falla: log + reintento. NO rollback.
PASO 4  return ServiceResponse<OnboardingResponseDto> { Status = "ONBOARDING_COMPLETE" }
        catch → RollbackAsync → throw (middleware → HTTP 400/500)
```

#### Endpoint

```
POST /Onboarding
Body:    OnboardingRequestDto
200 OK:  ServiceResponse<OnboardingResponseDto>
400:     ServiceResponse<null> { errorMessage: "[BR-ID] ..." }
500:     ServiceResponse<null>
```

#### Registro Program.cs

```csharp
builder.Services.AddScoped<IOnboardingService, OnboardingService>();
builder.Services.AddScoped<IResidentNotificationPort, EmailNotificationAdapter>();
```

#### Migración

```powershell
dotnet ef migrations add CU08_OnboardingDomain --output-dir Migrations
dotnet ef migrations script --output "scripts/002_CU08_OnboardingDomain.sql" --idempotent
dotnet ef database update
```

Tablas: `PB_Consorcio`, `PB_Complejo`, `PB_UnidadHabitacional`, `PB_Amenity`, `PB_AmenityConfig`

---

---

## SPEC-CU01
### CU-01: Reservar un Amenity

**Tipo:** Servicio específico (hereda `ServiceAsync<Reserva>`, override `Create`)
**Estado:** 🔲 Pendiente
**Migración:** `003_CU01_CU02_ReservaIncidencia` (compartida con CU-02)
**Entidades:** ENT-08 (Reserva)
**Actor:** Inquilino / Propietario
**Precondición:** sesión activa, unidad `EstadoUnidad = "ACTIVA"`, unidad `DebeExpensas = false`, amenity `Estado = "DISPONIBLE"`

#### Archivos

```
Models/               Reserva.cs
DTOs/CU01/            ReservaRequestDto.cs, ReservaResponseDto.cs
Services/ReservaService/    IReservaService.cs, ReservaService.cs
Controllers/          ReservaController.cs
```

#### DTOs

```csharp
// Input
public class ReservaRequestDto {
    public int      IDAmenity            { get; set; }   // Required
    public int      IDUnidadHabitacional { get; set; }   // Required
    public DateOnly FechaUso             { get; set; }   // Required
    public TimeOnly HoraInicio           { get; set; }   // Required
    public int      CantidadInvitados    { get; set; }   // default: 0
}

// Output
public class ReservaResponseDto {
    public int      IDReserva  { get; set; }
    public string   Estado     { get; set; }
    public DateOnly FechaUso   { get; set; }
    public TimeOnly HoraInicio { get; set; }
    public TimeOnly HoraFin    { get; set; }
    public string   NombreAmenity { get; set; }
    public string   IdentificadorUnidad { get; set; }
}
```

#### Business Rules — evaluación en ReservaService.Create, en este orden exacto

| BR-ID | P | RN | Condición | Acción |
|---|---|---|---|---|
| `BR-RES-001` | 1 | RN-01 | `UnidadHabitacional.DebeExpensas == true` | `BadRequestException("[BR-RES-001] La unidad tiene deuda de expensas pendiente.")` |
| `BR-RES-002` | 1 | RN-23 | `UnidadHabitacional.EstadoUnidad == "SUSPENDIDA"` | `BadRequestException("[BR-RES-002] La unidad se encuentra suspendida.")` |
| `BR-RES-003` | 1 | RN-08 | `HoraInicio < AmenityConfig.HorarioInicio` OR `HoraFin > AmenityConfig.HorarioFin` | `BadRequestException("[BR-RES-003] El bloque solicitado está fuera del horario operativo del amenity.")` |
| `BR-RES-004` | 1 | RN-10 | Fecha bloqueada por admin (tabla `PB_MantenimientoProgramado` o bloqueo especial) | `BadRequestException("[BR-RES-004] La fecha solicitada está bloqueada por la administración.")` |
| `BR-RES-005` | 1 | RN-06 | `FechaUso` fuera de `[now + H_horas, now + D_dias]` | `BadRequestException("[BR-RES-005] La reserva debe crearse con al menos {H} horas de anticipación y hasta {D} días.")` |
| `BR-RES-006` | 1 | RN-27 | `CantidadInvitados > Amenity.Capacidad` | `BadRequestException("[BR-RES-006] La cantidad de invitados supera la capacidad del amenity ({cap}).")` |
| `BR-RES-007` | 2 | RN-09 | Existe `Reserva` activa en mismo `IDAmenity`, misma `FechaUso`, con solapamiento horario | `BadRequestException("[BR-RES-007] El bloque horario solicitado ya está ocupado.")` |
| `BR-RES-008` | 2 | RN-03 | `COUNT(Reserva activa futura de la unidad) >= límite configurable` | `BadRequestException("[BR-RES-008] La unidad alcanzó el límite de reservas activas.")` |
| `BR-RES-009` | 3 | RN-02 | Unidad reservó mismo amenity >= 2 veces en mes actual | estado → `"EN_ESPERA"` (no excepción; continuar flujo) |
| `BR-RES-010` | 4 | RN-35 | `AmenityConfig.RequiereAprobacion == true` | estado → `"PENDIENTE_APROBACION"` |
| `BR-RES-011` | 5 | RN-13 | `AmenityConfig.Tarifa > 0` y modalidad pago previo | estado → `"PENDIENTE_PAGO"` |
| `BR-DEFAULT` | — | — | Ninguna condición anterior | estado → `"CONFIRMADA"` |

**Cálculo de HoraFin:** `HoraFin = HoraInicio + DuracionBloqueMinutos`. Calcular en el servicio antes de persistir.
**Solapamiento (BR-RES-007):** `HoraInicio_nueva < HoraFin_existente AND HoraFin_nueva > HoraInicio_existente`

#### Patrón de implementación — ReservaService

```csharp
public class ReservaService : ServiceAsync<Reserva> {
    // Inyectar además: IRepositoryAsync<Amenity>, IRepositoryAsync<UnidadHabitacional>,
    //                  IRepositoryAsync<AmenityConfig>, IRepositoryAsync<MantenimientoProgramado>
    public override async Task<Reserva> Create(Reserva entity) {
        // 1. Cargar Amenity + AmenityConfig + UnidadHabitacional
        // 2. Evaluar BRs P1 en orden de tabla
        // 3. Evaluar BR-RES-007 (solapamiento)
        // 4. Evaluar BR-RES-008 (límite activas)
        // 5. Calcular HoraFin, FechaSolicitud = DateTime.UtcNow
        // 6. Determinar Estado según BR-RES-009..011 y BR-DEFAULT
        // 7. await _repository.Insert(entity)
        // 8. Encolar notificación (RN-30) si Estado == "CONFIRMADA"
        // 9. return entity
    }
    protected override Expression<Func<Reserva, bool>> BuildCriterio(QueryParams qp) {
        // filtrar por IDAmenity y/o IDUnidadHabitacional si vienen en qp
    }
}
```

#### Endpoints (hereda GenericControllerAsync<Reserva> — override POST)

```
POST   /Reserva                   → Create con ReservaRequestDto
GET    /Reserva/GetById?id={id}   → GetById
GET    /Reserva/FindQP            → búsqueda paginada
PUT    /Reserva                   → Update (solo admin; estado CONFIRMADA → no modificable — RN-11)
DELETE /Reserva/{id}              → cancelar (validar margen — RN-05)
```

#### Registro Program.cs

```csharp
builder.Services.AddScoped<IServiceAsync<Reserva>, ReservaService>();
```

#### Migración

```powershell
dotnet ef migrations add CU01_CU02_ReservaIncidencia --output-dir Migrations
dotnet ef migrations script --output "scripts/003_CU01_CU02_ReservaIncidencia.sql" --idempotent
dotnet ef database update
```

---

---

## SPEC-CU02
### CU-02: Reportar Incidencia en Amenity

**Tipo:** Servicio específico (hereda `ServiceAsync<Incidencia>`, override `Create`)
**Estado:** 🔲 Pendiente
**Migración:** `003_CU01_CU02_ReservaIncidencia` (compartida con CU-01)
**Entidades:** ENT-09 (Incidencia)
**Actor:** Inquilino / Guardia / Administrador

#### Archivos

```
Models/               Incidencia.cs
DTOs/CU02/            IncidenciaRequestDto.cs, IncidenciaResponseDto.cs
Services/IncidenciaService/   IIncidenciaService.cs, IncidenciaService.cs
Controllers/          IncidenciaController.cs
```

#### DTOs

```csharp
// Input
public class IncidenciaRequestDto {
    public int    IDAmenity            { get; set; }   // Required
    public int    IDUnidadHabitacional { get; set; }   // Required
    public string Descripcion          { get; set; }   // Required
}

// Output
public class IncidenciaResponseDto {
    public int      IDIncidencia  { get; set; }
    public string   Estado        { get; set; }
    public string   NombreAmenity { get; set; }
    public DateTime FechaReporte  { get; set; }
    public string   Descripcion   { get; set; }
}
```

#### Business Rules — evaluación en IncidenciaService.Create, en este orden

| BR-ID | P | RN | Condición | Acción |
|---|---|---|---|---|
| `BR-INC-001` | 1 | RN-17 | Siempre al crear | `Amenity.Estado = "FUERA_DE_SERVICIO"` (update automático en la misma tx) |
| `BR-INC-002` | 2 | RN-18 | Tras BR-INC-001 | Cancelar en cascada todas las `Reserva` futuras del amenity (`Estado = "CANCELADA"`) |
| `BR-INC-003` | 3 | RN-15 | Por cada Reserva cancelada en cascada con `Tarifa > 0` | Acreditar monto a `UnidadHabitacional.SaldoActual` (o emitir crédito) |

#### Patrón de implementación — IncidenciaService

```csharp
public class IncidenciaService : ServiceAsync<Incidencia> {
    // Inyectar además: IRepositoryAsync<Amenity>, IRepositoryAsync<Reserva>,
    //                  IRepositoryAsync<AmenityConfig>, INotificationPort
    public override async Task<Incidencia> Create(Incidencia entity) {
        // Usar IDbContextTransaction (tx explícita — múltiples entidades afectadas)
        // 1. Cargar Amenity; validar que existe
        // 2. entity.Estado = "ABIERTA"; entity.FechaReporte = DateTime.UtcNow
        // 3. BEGIN TRANSACTION
        //    a) Insert Incidencia
        //    b) Amenity.Estado = "FUERA_DE_SERVICIO"; Update Amenity (BR-INC-001)
        //    c) Buscar Reservas futuras activas del amenity; foreach → Estado = "CANCELADA" (BR-INC-002)
        //    d) Por cada reserva cancelada con tarifa > 0: acreditar (BR-INC-003)
        //    e) COMMIT
        // 4. Notificar admin + unidades afectadas (RN-31) — fuera de tx
        // 5. return entity
    }
}
```

#### Endpoint

```
POST /Incidencia       → Create
GET  /Incidencia/GetById?id={id}
GET  /Incidencia/FindQP
```

#### Registro Program.cs

```csharp
builder.Services.AddScoped<IServiceAsync<Incidencia>, IncidenciaService>();
```

---

---

## SPEC-CU04
### CU-04: Resolver Incidencia y Rehabilitar Amenity

**Tipo:** Operación de actualización con efecto en cascada (método adicional en IncidenciaService)
**Estado:** 🔲 Pendiente
**Depende de:** SPEC-CU02 (misma entidad y servicio)
**Actor:** Administrador / Personal de Mantenimiento

#### DTO

```csharp
// DTOs/CU04/ResolucionDto.cs
public class ResolucionDto {
    public int     IDIncidencia    { get; set; }   // Required
    public string  DetalleTrabajos { get; set; }   // Required
    public decimal CostoEstimado   { get; set; }   // Required, min: 0
}
```

#### Business Rules — evaluación en IncidenciaService.Resolver

| BR-ID | P | RN | Condición | Acción |
|---|---|---|---|---|
| `BR-INC-004` | 1 | — | `Incidencia.Estado == "RESUELTA"` | `BadRequestException("[BR-INC-004] La incidencia ya fue resuelta.")` |
| `BR-INC-005` | 2 | RN-19 | Al cambiar a "RESUELTA" | `Amenity.Estado = "DISPONIBLE"` (automático, misma tx) |
| `BR-INC-006` | 3 | RN-21 | Siempre | Completar `Incidencia.DetalleResolucion`, `CostoEstimado`, `FechaResolucion = DateTime.UtcNow` |

#### Patrón de implementación

```csharp
// Agregar en IncidenciaService:
public async Task<ServiceResponse<IncidenciaResponseDto>> Resolver(ResolucionDto dto) {
    // BEGIN TRANSACTION
    // 1. Cargar Incidencia; evaluar BR-INC-004
    // 2. Incidencia.Estado = "RESUELTA"; DetalleResolucion, CostoEstimado, FechaResolucion — BR-INC-006
    // 3. Cargar Amenity; Amenity.Estado = "DISPONIBLE" — BR-INC-005
    // 4. Update Incidencia + Update Amenity
    // 5. COMMIT
    // return ServiceResponse<IncidenciaResponseDto>
}
```

#### Endpoint (agregar en IncidenciaController)

```
PUT /Incidencia/Resolver
Body:  ResolucionDto
200:   ServiceResponse<IncidenciaResponseDto>
400:   ServiceResponse<null>
```

---

---

## SPEC-CU03
### CU-03: Control de Acceso de Invitados

**Tipo:** Servicio específico (hereda `ServiceAsync<Invitado>`, endpoints adicionales propios)
**Estado:** 🔲 Pendiente
**Migración:** `004_CU03_InvitadoInquilino`
**Entidades:** ENT-07 (Invitado), ENT-06 (Inquilino)
**Actor:** Guardia de Seguridad / Recepción

#### Archivos

```
Models/               Invitado.cs, Inquilino.cs
DTOs/CU03/            AccesoConsultaDto.cs, AccesoResultadoDto.cs, EgresoDto.cs
Services/AccesoService/   IAccesoService.cs, AccesoService.cs
Controllers/          InvitadoController.cs   (hereda GenericControllerAsync<Invitado>)
                      AccesoController.cs      (controller propio para consulta/registro)
```

#### DTOs

```csharp
// Input consulta
public class AccesoConsultaDto { public string Dni { get; set; } }

// Output consulta
public class AccesoResultadoDto {
    public bool   Autorizado      { get; set; }
    public string Motivo          { get; set; }   // null si Autorizado
    public string UnidadAnfitriona { get; set; }  // null si denegado
    public int?   IDInvitado      { get; set; }
}

// Input egreso
public class EgresoDto { public int IDInvitado { get; set; } }
```

#### Business Rules — evaluación en AccesoService.Consultar

| BR-ID | P | RN | Condición | Acción |
|---|---|---|---|---|
| `BR-ACC-001` | 1 | — | `Invitado` con ese DNI no existe | `Autorizado = false; Motivo = "Invitado no registrado en ninguna unidad del consorcio."` |
| `BR-ACC-002` | 2 | RN-28 | `Invitado.EstadoAcceso == "DENEGADO"` | `Autorizado = false; Motivo = "Acceso denegado por historial negativo."` |
| `BR-DEFAULT` | — | — | Pasa todas las validaciones | `Autorizado = true; UnidadAnfitriona = identificador de la unidad` |

#### Patrón de implementación — AccesoService

```csharp
public interface IAccesoService {
    Task<AccesoResultadoDto> Consultar(string dni);
    Task RegistrarIngreso(int idInvitado);
    Task RegistrarEgreso(int idInvitado);     // RN-29
}

// Consultar:
//   1. Find Invitado por DNI
//   2. Evaluar BR-ACC-001
//   3. Evaluar BR-ACC-002
//   4. Registrar HoraIngreso = DateTime.UtcNow; Update Invitado
//   5. return AccesoResultadoDto

// RegistrarEgreso:
//   1. GetByID Invitado; si null → NotFoundException
//   2. Invitado.HoraEgreso = DateTime.UtcNow; Update
```

#### Endpoints

```
POST /Acceso/Consultar        Body: AccesoConsultaDto   → ServiceResponse<AccesoResultadoDto>
POST /Acceso/RegistrarIngreso Body: { IDInvitado: int } → ServiceResponse<object>
POST /Acceso/RegistrarEgreso  Body: EgresoDto           → ServiceResponse<object>    (RN-29)
GET  /Invitado/GetById?id={id}
GET  /Invitado/FindQP
POST /Invitado                → Create invitado (gestión desde unidad)
PUT  /Invitado                → Update (cambiar EstadoAcceso)
```

#### Registro Program.cs

```csharp
builder.Services.AddScoped<IAccesoService, AccesoService>();
builder.Services.AddScoped<IServiceAsync<Invitado>, ServiceAsync<Invitado>>();
builder.Services.AddScoped<IServiceAsync<Inquilino>, ServiceAsync<Inquilino>>();
```

#### Migración

```powershell
dotnet ef migrations add CU03_InvitadoInquilino --output-dir Migrations
dotnet ef migrations script --output "scripts/004_CU03_InvitadoInquilino.sql" --idempotent
```

Tablas: `PB_Invitado`, `PB_Inquilino`

---

---

## SPEC-CU05
### CU-05: Lista de Espera

**Tipo:** Servicio específico (hereda `ServiceAsync<ListaEspera>`, override `Create`)
**Estado:** 🔲 Pendiente
**Migración:** `005_CU05_ListaEspera`
**Entidades:** ENT-10 (ListaEspera)
**Actor:** Inquilino / Propietario

#### DTOs

```csharp
// Input
public class ListaEsperaRequestDto {
    public int      IDAmenity            { get; set; }
    public int      IDUnidadHabitacional { get; set; }
    public DateOnly FechaUso             { get; set; }
    public TimeOnly HoraInicio           { get; set; }
}

// Output
public class ListaEsperaResponseDto {
    public int    IDListaEspera { get; set; }
    public int    Posicion      { get; set; }
    public string Estado        { get; set; }
    public string NombreAmenity { get; set; }
    public DateOnly FechaUso   { get; set; }
}
```

#### Business Rules

| BR-ID | P | RN | Condición | Acción |
|---|---|---|---|---|
| `BR-ESP-001` | 1 | — | El bloque está disponible (no hay reserva activa) | `BadRequestException("[BR-ESP-001] El bloque solicitado está disponible; use el flujo de reserva normal.")` |
| `BR-ESP-002` | 2 | RN-02 | Unidad tiene >= 2 reservas del amenity en mes actual | `Posicion` se asigna después de las unidades con menor uso (fair-use) |
| `BR-DEFAULT` | — | — | — | `Posicion = MAX(Posicion) + 1`; `Estado = "ESPERANDO"` |

#### Flujo de liberación (trigger: Reserva cancelada)

```
Al cancelar Reserva:
  1. Buscar ListaEspera con mismo IDAmenity, FechaUso, HoraInicio, Estado = "ESPERANDO", ORDER BY Posicion ASC LIMIT 1
  2. Si existe: ListaEspera.Estado = "NOTIFICADO"; Notificar (RN-32) — tiempo límite T minutos
  3. Job/background: si transcurrido T y Estado sigue "NOTIFICADO" → Estado = "EXPIRADO"; pasar al siguiente
```

#### Registro Program.cs

```csharp
builder.Services.AddScoped<IServiceAsync<ListaEspera>, ListaEsperaService>();
```

#### Migración

```powershell
dotnet ef migrations add CU05_ListaEspera --output-dir Migrations
dotnet ef migrations script --output "scripts/005_CU05_ListaEspera.sql" --idempotent
```

---

---

## SPEC-CU06
### CU-06: Sancionar Unidad Habitacional

**Tipo:** Método adicional en servicio de UnidadHabitacional
**Estado:** 🔲 Pendiente
**Depende de:** ENT-03 (campo `EstadoUnidad`, `ContadorInfracciones` ya definidos en SPEC-ENT)
**Actor:** Administrador

#### DTOs

```csharp
// Input
public class SancionRequestDto {
    public int    IDUnidadHabitacional { get; set; }   // Required
    public string Descripcion          { get; set; }   // Required
    public bool   AplicarSuspension    { get; set; }   // true = suspender unidad
    public int    DuracionDias         { get; set; }   // 0 = indefinido
}

// Output
public class SancionResponseDto {
    public int    IDUnidadHabitacional { get; set; }
    public string EstadoUnidad         { get; set; }
    public int    ContadorInfracciones { get; set; }
    public string Mensaje              { get; set; }
}
```

#### Business Rules

| BR-ID | P | RN | Condición | Acción |
|---|---|---|---|---|
| `BR-SAN-001` | 1 | RN-26 | Siempre | `ContadorInfracciones++`; si >= umbral configurable → notificar admin |
| `BR-SAN-002` | 2 | RN-23 | `AplicarSuspension == true` | `EstadoUnidad = "SUSPENDIDA"`; cancelar reservas futuras activas de la unidad |
| `BR-SAN-003` | 3 | RN-31 | `AplicarSuspension == true` | Notificar a residentes de la unidad con motivo y duración |

#### Patrón de implementación

```csharp
// Agregar en UnidadHabitacionalService (o servicio orquestador propio):
public async Task<SancionResponseDto> Sancionar(SancionRequestDto dto) {
    // BEGIN TRANSACTION
    // 1. Cargar UnidadHabitacional; null → NotFoundException
    // 2. ContadorInfracciones++ (BR-SAN-001)
    //    Si ContadorInfracciones >= umbral → loguear alerta para admin
    // 3. Si AplicarSuspension:
    //    EstadoUnidad = "SUSPENDIDA"
    //    Buscar Reservas activas futuras de la unidad → Estado = "CANCELADA" (BR-SAN-002)
    // 4. Update UnidadHabitacional
    // 5. COMMIT
    // 6. Notificar residentes (BR-SAN-003) — fuera de tx
}
```

#### Endpoint

```
POST /UnidadHabitacional/Sancionar
Body: SancionRequestDto → ServiceResponse<SancionResponseDto>
```

---

---

## SPEC-CU07
### CU-07: Pago de Reserva

**Tipo:** Servicio orquestador propio (NO hereda `ServiceAsync<T>`)
**Estado:** 🔲 Pendiente
**Depende de:** SPEC-CU01
**Actor:** Inquilino / Propietario

#### Archivos

```
Services/Ports/     IPaymentGatewayPort.cs
Infrastructure/Adapters/    PaymentGatewayAdapter.cs
Services/PagoService/       IPagoService.cs, PagoService.cs
Controllers/        PagoController.cs
```

#### DTOs

```csharp
// Input
public class PagoRequestDto {
    public int    IDReserva      { get; set; }   // Required
    public string MetodoPago     { get; set; }   // "TARJETA" | "TRANSFERENCIA" | "BILLETERA_DIGITAL"
    public string TokenPasarela  { get; set; }   // token generado por el frontend desde el SDK de la pasarela
}

// Output
public class PagoResponseDto {
    public int      IDReserva     { get; set; }
    public string   EstadoReserva { get; set; }   // "CONFIRMADA" o "CANCELADA"
    public bool     PagoExitoso   { get; set; }
    public string   Comprobante   { get; set; }   // URL o número de comprobante (RN-16)
}
```

#### Business Rules

| BR-ID | P | RN | Condición | Acción |
|---|---|---|---|---|
| `BR-PAG-001` | 1 | — | `Reserva.Estado != "PENDIENTE_PAGO"` | `BadRequestException("[BR-PAG-001] La reserva no está en estado de pago pendiente.")` |
| `BR-PAG-002` | 2 | RN-13 | Pago exitoso en pasarela | `Reserva.Estado = "CONFIRMADA"`; emitir comprobante (RN-16) |
| `BR-PAG-003` | 2 | RN-13 | Pago fallido | `Reserva.Estado = "CANCELADA"`; liberar bloque |
| `BR-PAG-004` | 3 | RN-14 | Amenity de tipo evento (SUM/Parrilla) | Adicionar depósito de garantía al monto |
| `BR-PAG-005` | — | RN-16 | Cualquier transacción exitosa | Emitir comprobante digital (PDF/email) |

#### Port interface

```csharp
public interface IPaymentGatewayPort {
    Task<(bool Exitoso, string Referencia)> ProcesarPagoAsync(string token, decimal monto, string metodo);
}
```

#### Registro Program.cs

```csharp
builder.Services.AddScoped<IPagoService, PagoService>();
builder.Services.AddScoped<IPaymentGatewayPort, PaymentGatewayAdapter>();
```

#### Endpoint

```
POST /Pago
Body: PagoRequestDto → ServiceResponse<PagoResponseDto>
```

---

---

## SPEC-CU09
### CU-09: Reportes y Auditoría

**Tipo:** Servicio de solo lectura + AuditLog cross-cutting
**Estado:** 🔲 Pendiente
**Migración:** `006_CU09_AuditLog`
**Entidades:** ENT-12 (AuditLog)
**Actor:** Administrador

#### Archivos

```
Models/               AuditLog.cs
DTOs/CU09/            ReporteRequestDto.cs, ReporteResponseDto.cs, AuditLogDto.cs
Services/ReporteService/    IReporteService.cs, ReporteService.cs
Services/              IAuditService.cs, AuditService.cs   ← cross-cutting; inyectado en servicios que emiten acciones críticas
Controllers/          ReporteController.cs
```

#### DTOs

```csharp
// Input filtros
public class ReporteRequestDto {
    public string   TipoReporte { get; set; }   // "USO_AMENITIES" | "MOROSIDAD" | "INCIDENCIAS" | "INGRESOS"
    public DateOnly FechaDesde  { get; set; }
    public DateOnly FechaHasta  { get; set; }
    public int?     IDAmenity   { get; set; }   // nullable → todos
    public int?     IDComplejo  { get; set; }   // nullable → todos
}

// Output genérico (el agente adaptará Data según TipoReporte)
public class ReporteResponseDto {
    public string       TipoReporte { get; set; }
    public DateOnly     FechaDesde  { get; set; }
    public DateOnly     FechaHasta  { get; set; }
    public List<object> Data        { get; set; }
    public int          TotalRegistros { get; set; }
}
```

#### Business Rules

| BR-ID | RN | Regla |
|---|---|---|
| `BR-AUD-001` | RN-38 | `AuditService.Registrar(usuario, accion, entidad, id, detailJson)` debe invocarse desde: `ReservaService.Create`, `ReservaService.Delete`, `IncidenciaService.Create`, `IncidenciaService.Resolver`, `SancionarUnidad`, toda modificación de `AmenityConfig`. |
| `BR-AUD-002` | RN-40 | Reporte MOROSIDAD: consulta directa sin caché. Retornar unidades con `DebeExpensas = true` que tengan `Reserva` activa futura. |

#### IAuditService interface

```csharp
public interface IAuditService {
    Task Registrar(string usuario, string accion, string entidad, int entidadId, object detalle);
}
// AuditService implementa: serializa detalle a JSON, Insert AuditLog vía IRepositoryAsync<AuditLog>
```

#### Registro Program.cs

```csharp
builder.Services.AddScoped<IReporteService, ReporteService>();
builder.Services.AddScoped<IAuditService, AuditService>();
builder.Services.AddScoped<IServiceAsync<AuditLog>, ServiceAsync<AuditLog>>();
```

#### Migración

```powershell
dotnet ef migrations add CU09_AuditLog --output-dir Migrations
dotnet ef migrations script --output "scripts/006_CU09_AuditLog.sql" --idempotent
```

---

---

## SPEC-CU10
### CU-10: Mantenimiento Programado Preventivo

**Tipo:** Servicio específico (hereda `ServiceAsync<MantenimientoProgramado>`, override `Create`)
**Estado:** 🔲 Pendiente
**Migración:** `007_CU10_MantenimientoProgramado`
**Entidades:** ENT-11 (MantenimientoProgramado)
**Actor:** Administrador

#### DTOs

```csharp
// Input
public class MantenimientoRequestDto {
    public int      IDAmenity   { get; set; }    // Required
    public string   Descripcion { get; set; }    // Required
    public string   Recurrencia { get; set; }    // Required: "LUNES".."DOMINGO" | "DIARIO" | "SEMANAL"
    public string   HoraInicio  { get; set; }    // Required "HH:mm"
    public string   HoraFin     { get; set; }    // Required "HH:mm" > HoraInicio
    public DateOnly FechaInicio { get; set; }    // Required
    public DateOnly FechaFin    { get; set; }    // Required > FechaInicio
}

// Output
public class MantenimientoResponseDto {
    public int      IDMantenimiento { get; set; }
    public string   NombreAmenity   { get; set; }
    public string   Recurrencia     { get; set; }
    public string   HoraInicio      { get; set; }
    public string   HoraFin         { get; set; }
    public DateOnly FechaInicio     { get; set; }
    public DateOnly FechaFin        { get; set; }
    public int      ReservasCanceladas { get; set; }   // cantidad canceladas en cascada
}
```

#### Business Rules

| BR-ID | P | RN | Condición | Acción |
|---|---|---|---|---|
| `BR-MAN-001` | 1 | — | `HoraFin <= HoraInicio` | `BadRequestException("[BR-MAN-001] HoraFin debe ser posterior a HoraInicio.")` |
| `BR-MAN-002` | 1 | — | `FechaFin <= FechaInicio` | `BadRequestException("[BR-MAN-002] FechaFin debe ser posterior a FechaInicio.")` |
| `BR-MAN-003` | 2 | RN-20 | Al crear mantenimiento | Buscar Reservas en los bloques afectados; cancelarlas en cascada (BR-MAN-004) |
| `BR-MAN-004` | 3 | RN-31 / RN-15 | Por cada Reserva cancelada | Notificar afectados; si `Tarifa > 0` → acreditar (igual que BR-INC-003) |

#### Registro Program.cs

```csharp
builder.Services.AddScoped<IServiceAsync<MantenimientoProgramado>, MantenimientoService>();
```

#### Migración

```powershell
dotnet ef migrations add CU10_MantenimientoProgramado --output-dir Migrations
dotnet ef migrations script --output "scripts/007_CU10_MantenimientoProgramado.sql" --idempotent
```

---

---

## SPEC-CU11
### CU-11: Baja Inquilino / Cambio de Ocupante

**Tipo:** Método adicional en InquilinoService
**Estado:** 🔲 Pendiente
**Depende de:** ENT-06 (Inquilino — campo `Activo` ya definido)
**Actor:** Administrador / Propietario

#### DTO

```csharp
// Input
public class BajaInquilinoDto { public int IDInquilino { get; set; } }

// Output
public class BajaInquilinoResponseDto {
    public int    IDInquilino          { get; set; }
    public int    IDUnidadHabitacional { get; set; }
    public bool   AccesoRevocado       { get; set; }
    public int    ReservasVinculadas   { get; set; }   // reservas que quedan vinculadas a la unidad
}
```

#### Business Rules

| BR-ID | P | RN | Condición | Acción |
|---|---|---|---|---|
| `BR-BAJ-001` | 1 | RN-44 | Siempre | `Inquilino.Activo = false`; invalidar token JWT del inquilino (tabla `PB_TokenRevocado`) |
| `BR-BAJ-002` | 2 | RN-44 | Las `Reserva` de la unidad | Permanecen asociadas a `IDUnidadHabitacional` (NO al `IDInquilino`); no se cancelan automáticamente |

#### Patrón de implementación

```csharp
// Agregar en InquilinoService:
public async Task<BajaInquilinoResponseDto> DarDeBaja(int idInquilino) {
    // 1. Cargar Inquilino; null → NotFoundException
    // 2. Inquilino.Activo = false; Update (BR-BAJ-001)
    // 3. Insert en PB_TokenRevocado para invalidar JWT activos del inquilino
    // 4. Contar Reserva activas futuras de la UnidadHabitacional (informativo — BR-BAJ-002)
    // 5. return BajaInquilinoResponseDto
}
```

#### Endpoint

```
POST /Inquilino/DarDeBaja
Body: BajaInquilinoDto → ServiceResponse<BajaInquilinoResponseDto>
```

---

---

## SPEC-NOTIF
### Notificaciones y Comunicaciones

**Tipo:** Cross-cutting — Port & Adapter
**Estado:** 🔲 Pendiente

#### Interface (Core)

```csharp
// Services/Ports/INotificationPort.cs
public interface INotificationPort {
    Task EnviarAsync(NotificacionDto notif);
}

public class NotificacionDto {
    public List<string> Destinatarios { get; set; }   // emails o device tokens
    public string       Canal         { get; set; }   // "EMAIL" | "PUSH" | "AMBOS"
    public string       Asunto        { get; set; }
    public string       Cuerpo        { get; set; }
    public string       TipoEvento    { get; set; }   // BR-NOT-ID de referencia
}
```

#### Notificaciones requeridas por CU

| BR-ID | RN | Trigger | Canal | Implementado en |
|---|---|---|---|---|
| `BR-NOT-001` | RN-30 | Reserva confirmada (job X horas antes) | Push + Email | Job background |
| `BR-NOT-002` | RN-31 | Cancelación automática por sistema | Push + Email | ReservaService, IncidenciaService, MantenimientoService |
| `BR-NOT-003` | RN-32 | Turno liberado → primera posición en lista espera | Push | ListaEsperaService |
| `BR-NOT-004` | RN-33 | Comunicado masivo del administrador | Push + Email | Endpoint propio en ReporteController o NotificacionController |

#### Registro Program.cs

```csharp
builder.Services.AddScoped<INotificationPort, NotificationAdapter>();
builder.Services.AddScoped<IResidentNotificationPort, EmailNotificationAdapter>();
```

---

---

## SPEC-AUTH
### Autenticación, Roles y Sesiones

**Tipo:** Extensión del sistema JWT existente en ProyectoBase
**Estado:** 🔲 Pendiente
**Migración:** `008_CU_Auth_Usuarios`
**Base:** `AccountController` + `TokenService` + `appsettings.json:Jwt` ya implementados

#### Roles del sistema

| Rol (string en claim) | Permisos |
|---|---|
| `RESIDENTE` | POST /Reserva, GET propio historial, POST /Incidencia, gestión de invitados propios |
| `GUARDIA` | POST /Acceso/Consultar, POST /Acceso/RegistrarIngreso, POST /Acceso/RegistrarEgreso — solo lectura en resto |
| `ADMINISTRADOR` | Acceso completo + sanciones + reportes + configuración de amenities |
| `SUPER_ADMINISTRADOR` | Igual que ADMIN + gestión multi-consorcio (CU-08 habilitado solo para este rol) |

#### Entidades nuevas requeridas

```csharp
// Models/Usuario.cs
public class Usuario {
    public int    IDUsuario           { get; set; }
    public int    IDUnidadHabitacional { get; set; }   // FK; nullable para ADMIN/GUARDIA
    public string Email               { get; set; }   // único; usado como login
    public string PasswordHash        { get; set; }
    public string Rol                 { get; set; }   // "RESIDENTE" | "GUARDIA" | "ADMINISTRADOR" | "SUPER_ADMINISTRADOR"
    public bool   Activo              { get; set; }   // default: true
    public UnidadHabitacional UnidadHabitacional { get; set; }
}

// Models/TokenRevocado.cs  — para BR-BAJ-001 (baja inquilino) y BR-AUTH-003
public class TokenRevocado {
    public int      IDTokenRevocado { get; set; }
    public string   Jti             { get; set; }   // JWT ID claim — único
    public DateTime Expiracion      { get; set; }   // UTC; para limpieza periódica
}
```

```csharp
// Fluent API — Usuario
modelBuilder.Entity<Usuario>(e => {
    e.ToTable("PB_Usuario");
    e.HasKey(x => x.IDUsuario);
    e.Property(x => x.Email).IsRequired().HasMaxLength(100);
    e.HasIndex(x => x.Email).IsUnique();
    e.Property(x => x.PasswordHash).IsRequired().HasMaxLength(256);
    e.Property(x => x.Rol).IsRequired().HasMaxLength(25);
    e.Property(x => x.Activo).HasDefaultValue(true);
    e.HasOne(x => x.UnidadHabitacional).WithMany()
     .HasForeignKey(x => x.IDUnidadHabitacional).IsRequired(false).OnDelete(DeleteBehavior.Restrict);
});

// Fluent API — TokenRevocado
modelBuilder.Entity<TokenRevocado>(e => {
    e.ToTable("PB_TokenRevocado");
    e.HasKey(x => x.IDTokenRevocado);
    e.Property(x => x.Jti).IsRequired().HasMaxLength(100);
    e.HasIndex(x => x.Jti).IsUnique();
    e.Property(x => x.Expiracion).IsRequired().HasColumnType("timestamptz");
});
```

#### Business Rules

| BR-ID | RN | Regla de implementación |
|---|---|---|
| `BR-AUTH-001` | RN-41 | `AccountController.Login` valida email+password contra `PB_Usuario`; retorna JWT con claims: `ClaimTypes.Name = email`, `ClaimTypes.Role = rol`, `JwtRegisteredClaimNames.Jti = Guid.NewGuid()` |
| `BR-AUTH-002` | RN-42 | Decorar cada endpoint con `[Authorize(Roles = "ROL1,ROL2")]` según tabla de roles |
| `BR-AUTH-003` | RN-43 | Middleware de validación consulta `PB_TokenRevocado` por `Jti` en cada request autenticado; si existe → `401` |
| `BR-AUTH-004` | RN-44 | Al ejecutar `DarDeBaja` (CU-11) → Insert en `PB_TokenRevocado` con todos los `Jti` activos del usuario |

#### Registro Program.cs

```csharp
builder.Services.AddScoped<IServiceAsync<Usuario>, ServiceAsync<Usuario>>();
builder.Services.AddScoped<IServiceAsync<TokenRevocado>, ServiceAsync<TokenRevocado>>();
```

#### Migración

```powershell
dotnet ef migrations add Auth_UsuariosTokenRevocado --output-dir Migrations
dotnet ef migrations script --output "scripts/008_Auth_UsuariosTokenRevocado.sql" --idempotent
```

---

## Orden de Implementación por Fases

```
Fase 1 — Base (ya completada)
  SPEC-ARCH  →  SPEC-ENT (ENT-01..05)  →  SPEC-CU08

Fase 2 — Auth + Core funcional
  SPEC-AUTH  →  SPEC-CU01  →  SPEC-CU02  →  SPEC-CU04

Fase 3 — Operaciones avanzadas
  SPEC-CU03  →  SPEC-CU05  →  SPEC-CU06  →  SPEC-CU07

Fase 4 — Administración y reporting
  SPEC-CU09  →  SPEC-CU10  →  SPEC-CU11  →  SPEC-NOTIF
```

