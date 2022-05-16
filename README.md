### Выбор между static и non static методами

В большинстве случаев используются обычные методы экземпляра, но иногда все же стоит использовать `static`.

Преимущества класса со `static` методами в том, что у нас нет необходимости создавать объекты через оператор `new`, нет
необходимости внедрять зависимости (dependency injection) в разных классах проекта, тем самым создавая тесные связи
между объектами, и при этом мы не забиваем память большим количеством абсолютно одинаковых объектов, создаваемых через
оператор new.  
Класс со `static` методами будет единственным на весь проект, можно сказать универсальным. А если предполагается, что у
нас может быть множество экземпляров класса с разными состояниями, то лучше использовать обычные методы.  
При этом `static` может стать причиной неочевидных багов в сложных проектах.

В случае, если не планируется хранить разные состояния объекта, можно использовать статические поля, например:
<details>
<summary>Класс с методами конвертации форматов:</summary>

```java
class Converter {
    public static JSONObject convertStringToJsonObject(String json) {
        return new JSONObject(json);
    }

    public static String convertJSONObjectToString(JSONObject json) {
        return json.toString();
    }
}

class Main {
    public static void main(String[] args) {
        //Использование
        var json = Converter.convertStringToJsonObject(jsonStr);
        var str = Converter.convertJSONObjectToString(jsonObj);
        // Методы не зависят друг от друга, нет ни одного поля класса, при создании объекта состояние будет неизменное, поэтому смысла создавать объект класса нет
    }
}

```

</details>  
<details>
<summary>Простой класс для работы с sql, где не используется Hibernate или другие ORM:</summary>

```java
class DbUtils {
    private static connection;

    static {
        var url = System.getProperty("db.url");
        var login = System.getProperty("db.login");
        var pass = System.getProperty("db.pass");
        connection = DriverManager.getConnection(url, login, pass);
    }

    public static long getUsersCount() {
        var query = "select count(*) from some_table";
        var count = connection.executeQuery(query);
        returt count;
    }

    public static String getPasswordByLogin(String login) {
        var query = "select password from some_table where login = login";
        var password = connection.executeQuery(query);
        returt password;
    }
}

class Main {
    public static void main(String[] args) {
        // Вместо того, чтобы создавать множество объектов через оператор new DbUtils() в разных классах и хранить в памяти несколько одинаковых соединений
        // можно просто использовать статические методы, вызывая их непосредственно из класса и тем самым использовать единственное соединение с БД
        var usersCount = DbUtils.getUsersCount();
        var userPass = DbUtils.getPasswordByLogin("login");
        // Методы зависят только от соединения,
        // соединение создается единожды в статическом блоке и используется до завершения работы кода
    }
}
```

</details>  
<details>
<summary>Классы для генерации тестовых данных</summary>

```java
class DataGenerator {
    public static int getRandomNum(int min, int max) {
        return RandomUtils.nextInt(min, max);
    }

    public static String getRandomString(int length) {
        return RandomStringUtils.random(length);
    }
}

class Main {
    public static void main(String[] args) {
        // Использование
        var randomNum = DataGenerator.getRandomNum(1, 10);
        var randomStr = DataGenerator.getRandomString(10);
    }
}
```

</details>  

### Когда лучше не использовать статические методы:

<details>
<summary>Пример 1</summary>

```java

@Data
class Employee {
    private String name;
    private float salary;
    private int childrenCount;
    private final float CHILDREN_DISCOUNT_PERCENT = 1.1f;

    public float calculateBonus(int bonusPercent) {
        return salary / 100 * bonusPercent;
    }

    public float calculateChildrenDiscount() {
        return CHILDREN_DISCOUNT_PERCENT * childrenCount;
    }
}

class Main {
    public static void main(String[] args) {
        // Предполагается, что объектов сотрудников будет много, у них будут разные внутренние состояния
        Employee firstEmployee = new Employee("Nick", 100.5, 0);
        Employee secondEmployee = new Employee("John", 70.2, 2);
        var firstDiscount = firstEmployee.calculateChildrenDiscount();
        var secondDiscount = secondEmployee.calculateChildrenDiscount();
        // объекты с разным состоянием, для каждого необъодимо выполнить множество разных методов, используемых эти состояния
        // в случае со статическими методами пришлось бы передавать в каждый метод одни и те же значения для расчетов

    }
}
```

</details>  
<details>
<summary>Пример 2</summary>

```java

@Data
class WorkProcess {
    private String processName;
    private Stage currentStage;
    private boolean isUserActiveNow;

    public String getStage() {
        return currentStage.getName();
    }

    public void goNewStage(Stage newStage) {
        this.currentStage = newStage;
    }

    public void stopProcess() {
        this.isUserActiveNow = false;
        Dashboard.pushCurrentStatus(this);
    }
}
// Разные объекты рабочих процессов в различном состоянии, 
// управляются независимо друг друга, но внутри объекта все поля и методы тесно связаны между собой
```

</details>
<details>
<summary>Пример 3</summary>

```java
class Employee {
    private String name;
    // Здесь не нужно использовать static
    private static String serialNumber;

    public Employee(String name) {
        this.name = name;
    }

    // При такой реализации мы присвоим номер не конкретному сотруднику, а всем сотрудникам сразу, 
    // потому что статическое поле будет иметь одно и то же значение во всех объектах Employee, 
    // так как статическое поле будет единственным, несмотря на количество созданных Employee
    public void setSerialNumber(int serialNumber) {
        return this.serialNumber = serialNumber;
    }

    public int getSerialNumber() {
        return serialNumber;
    }
}

class Main {
    public static void main(String[] args) {
        // 
        Employee firstEmployee = new Employee("Nick");
        firstEmployee.setSerialNumber("employee_1");
        Employee secondEmployee = new Employee("John");
        secondEmployee.setSerialNumber("employee_2");

        System.out.println(firstEmployee.getSerialNumber());
        System.out.println(secondEmployee.getSerialNumber());
        // Оба вывода в консоль выведут значение 'employee_2', 
        // так как статическая переменная serialNumber не может быть уникальной для всех созданных объектов Employee,
        // она будет иметь сначала значение 'employee_1', а потом 'employee_2' одновременно для всех объектов, созданных через new Employee()
    }
}
```

</details>
