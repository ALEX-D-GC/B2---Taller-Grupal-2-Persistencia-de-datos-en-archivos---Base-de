# B2---Taller-Grupal-2-Persistencia-de-datos-en-archivos---Base-de
### 1. Configuración de la Base de Datos (Archivo: config/Database.scala)
```scala
package config
import cats.effect.{IO, Resource}
import com.typesafe.config.ConfigFactory
import doobie.hikari.HikariTransactor
import scala.concurrent.ExecutionContext
object Database {
  private val connectEC: ExecutionContext = ExecutionContext.global
  def transactor: Resource[IO, HikariTransactor[IO]] = {
    val config = ConfigFactory.load().getConfig("db")
    HikariTransactor.newHikariTransactor[
      IO
    ](
      config.getString("driver"),
      config.getString("url"),
      config.getString("user"),
      config.getString("password"),
      connectEC // ExecutionContext requerido para Doobie
    )
  }
}
```
### 2  DAO para la Interacción con la Base de Datos (Archivo: dao/PersonasDAO.scala)
```scala
package dao
import cats.effect.IO
import cats.implicits._
import config.Database
import doobie._
import doobie.implicits._
import models.Estudiante
object PersonasDAO {
  // Método para insertar un estudiante
  def insert(estudiante: Estudiante): ConnectionIO[Int] = {
    sql"""
      INSERT INTO estudiantes (nombre, edad, calificacion, genero)
      VALUES (
        ${estudiante.nombre},
        ${estudiante.edad},
        ${estudiante.calificacion},
        ${estudiante.genero}
      )
    """.update.run
  }
  // Método para insertar una lista de estudiantes
  def insertAll(estudiantes: List[Estudiante]): IO[List[Int]] = {
    Database.transactor.use { xa =>
      estudiantes.traverse(e => insert(e).transact(xa))
    }
  }
  // **Nuevo método**: Obtener todos los estudiantes de la base de datos
  def getAll: IO[List[Estudiante]] = {
    val query = sql"""
      SELECT nombre, edad, calificacion, genero
      FROM estudiantes
    """.query[Estudiante].to[List] // Ejecuta la consulta y mapea a una lista de Estudiantes
    Database.transactor.use { xa =>
      query.transact(xa)
    }
  }
}
```
### 3. Modelo para los Personas
```scala
package models
// Modelo para la tabla "estudiantes"
case class Estudiante(
                       nombre: String,       // Nombre del estudiante
                       edad: Int,            // Edad del estudiante
                       calificacion: Int,    // Calificación del estudiante
                       genero: String        // Género del estudiante ('M' o 'F')
                     )
```
### 4. Función Principal (Archivo: Main.scala)
```scala
import cats.effect.{IO, IOApp}
import kantan.csv._
import kantan.csv.ops._
import kantan.csv.generic._
import java.io.File
import models.Estudiante
import dao.PersonasDAO
import cats.implicits._ // Esto añade métodos como `traverse` a las colecciones
object Main extends IOApp.Simple {
  val path2DataFile2 = "/Users/monky/NetBeansProjects/personas/src/main/resources/data/people.csv"
  val dataSource = new File(path2DataFile2)
    .readCsv[List, Estudiante](rfc.withHeader.withCellSeparator(','))
  val estudiantes = dataSource.collect {
    case Right(estudiante) => estudiante
  }
  // Secuencia de operaciones IO usando for-comprehension
  def run: IO[Unit] = for {
    // Inserta todos los registros en la base de datos
    insertResult <- PersonasDAO.insertAll(estudiantes)
    _ <- IO.println(s"Registros insertados: ${insertResult.size}")
    // Obtiene todos los registros y los imprime
    allStudents <- PersonasDAO.getAll
    _ <- allStudents.traverse(s => IO.println(s))
  } yield ()
}
```


### 5. Configuracion

```Scala
db {
  driver = "com.mysql.cj.jdbc.Driver"
  url = "jdbc:mysql://localhost:3306/estudiantes"
  user = "root"
  password = ""
}
```

### 6. CSV

nombre,edad,calificacion,genero
Andrés,10,20,M
Ana,11,19,F
Luis,9,18,M
Cecilia,9,18,F
Katy,11,15,F
Jorge,8,17,M
Rosario,11,18,F
Nieves,10,20,F
Pablo,9,19,M
Daniel,10,20,M


