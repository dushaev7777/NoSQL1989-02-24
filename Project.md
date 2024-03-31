# Задание #

Взять 2+ NoSQL БД, загрузить 10+ млн записей и провести масштабное исследование по скорости обработки запросов.

1. Цели проекта
2. Что планировалось
3. Используемые технологии
4. Схемы/архитектура
5. Что получилось
6. Выводы
7. шаблон презентации в ЛК 15-20 минут

# Решение #

1. Целью проекта является сравнение производительности при вставке 10млн записей в СУБД NoSQL MongoDB и Redis.
2. Планируется загрузить в СУБД 10 млн записей с помощью Java запросов, а также провести поиск, обновление и удаление данных из СУБД, по полученным данным посмотреть какая СУБД справиться эффективней с поставленной задачей.
3. Используемые технологии:
   * Hyper-V
   * IntelliJ IDEA 2023.3.2
   * Maven

4. СУБД будут установлены на ВМ под управлением Hyper-V с ОС Linux, имеющие одинаковые параметры:
   * Процессор — 2,1 ГГц (2 ядра)
   * Память - 4 ГБ
   * Хранилище - 10 ГБ
   * Операционная система — CentOS 7.9
   * Версии СУБД: MongoDB 7.0.5, Redis 7.2.3(использованы последние версии по предложению на февраль 2024 года)

Запросы Java будут отправляться через IntelliJ IDEA с использованием инструмента Maven для сборки проекта, он же будет являться сервером приложения.

Запросы сгенирорированы максимально близко логически к разным СУБД.

Скорость выполнения Java запросов будем измерять с помощью Profiler IntelliJ IDEA, чтобы получить временные значения приближенные к работе приложения.

5. СУБД установлены и настроены, производим настройку проектов IntelliJ IDEA, с последующей отправкой запросов.

#### 5.1 MongoDB

Настраиваем проект IntelliJ IDEA:

![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Mongo_1.png)

Добавляем зависимость Mongo драйвера в pom.xml файл:
```
    <dependencies>
        <dependency>
            <groupId>org.mongodb</groupId>
            <artifactId>mongo-java-driver</artifactId>
            <version>3.12.14</version>
        </dependency>
    </dependencies>
```
![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Mongo_2.png)

Настроим подключение к базе данных MongoDB в IntelliJ IDEA:

![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Mongo_3.png)

Запустим Java запрос на генерацию 10млн строк и вставку в БД построчно:
```
import com.mongodb.MongoClient;
import com.mongodb.client.MongoDatabase;
import com.mongodb.client.MongoCollection;
import org.bson.Document;

public class Main {
    public static void main(String[] args) {
        String host = "192.168.0.115";
        String user = "admin";
        String passw = "admin";
        int port = 27017;
        String dbName = "test_db";
        String collectionName = "test_coll";

        MongoClient mongoClient = new MongoClient(host, port);
        MongoDatabase database = mongoClient.getDatabase(dbName);
        MongoCollection<Document> collection = database.getCollection(collectionName);

        for (int i = 0; i < 10000000; i++) {
            Document document = new Document("id", i).append("name", "Document " + i);
            collection.insertOne(document);
        }

        System.out.println("10 million documents inserted successfully.");

        mongoClient.close();
    }
}
```
![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Mongo_4.png)

Проверим данные в подключенной БД MongoDB:

![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Mongo_s1.png)

Запрос на генерацию 10млн записей и вставки в виде массива данных:
```
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoDatabase;
import com.mongodb.client.MongoCollection;
import org.bson.Document;
import java.util.ArrayList;
import java.util.List;

public class Main {
    public static void main(String[] args) {
        String host = "192.168.0.115";
        String user = "admin";
        String passw = "admin";
        int port = 27017;
        String dbName = "test_db";
        String collectionName = "test_coll";

        MongoClient mongoClient = MongoClients.create("mongodb://192.168.0.115:27017");
        MongoDatabase database = mongoClient.getDatabase(dbName);
        MongoCollection<Document> collection = database.getCollection(collectionName);

        List<Document> documents = new ArrayList<>();
        for (int i = 0; i < 10000000; i++) {
            Document document = new Document("id", i).append("name", "Document " + i);
            documents.add(document);
        }
        collection.insertMany(documents);

        System.out.println("10 million documents inserted successfully.");

        mongoClient.close();
    }
}
```
Столкнулся с ошибкой java.lang.OutOfMemoryError: Java heap space:

![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Mongo_5.png)

Увеличил количество памяти на сервере приложения в настройка запуска запроса до 8ГБ:

![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Mongo_6.png)

Запустил повторно запрос:

![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Mongo_7.png)

Отправим запрос на выборку данных с условием что в документах будет встречаться число 999:
```
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.MongoCursor;
import com.mongodb.client.MongoDatabase;
import org.bson.Document;

import java.util.regex.Pattern;

public class Main {
    public static void main(String[] args) {
        MongoClient mongoClient = MongoClients.create("mongodb://192.168.0.115:27017");
        MongoDatabase database = mongoClient.getDatabase("test_db");
        MongoCollection<Document> collection = database.getCollection("test_coll");

        Pattern pattern = Pattern.compile(".*999.*");
        Document query = new Document("name", pattern);

        MongoCursor<Document> cursor = collection.find(query).iterator();

        while (cursor.hasNext()) {
            Document document = cursor.next();
            System.out.println(document.toJson());
        }

        cursor.close();
    }
}
```
![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Mongo_8.png)

Произведем удаление записей где встречается в поле "name" число 999:
```
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.MongoDatabase;
import org.bson.Document;

import java.util.regex.Pattern;

public class Main {
    public static void main(String[] args) {
        MongoClient mongoClient = MongoClients.create("mongodb://192.168.0.115:27017");
        MongoDatabase database = mongoClient.getDatabase("test_db");
        MongoCollection<Document> collection = database.getCollection("test_coll");

        Pattern pattern = Pattern.compile(".*999.*");
        Document query = new Document("name", pattern);

        collection.deleteMany(query);

        System.out.println("Documents with 'name' containing '999' deleted successfully.");
    }
}
```
![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Mongo_9.png)

Обновление документов с числом 777 на 999:
```
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.MongoCursor;
import com.mongodb.client.MongoDatabase;
import org.bson.Document;

import java.util.regex.Pattern;

public class Main {
    public static void main(String[] args) {
        MongoClient mongoClient = MongoClients.create("mongodb://192.168.0.115:27017");
        MongoDatabase database = mongoClient.getDatabase("test_db");
        MongoCollection<Document> collection = database.getCollection("test_coll");

        Pattern pattern = Pattern.compile(".*777.*");
        Document query = new Document("name", pattern);

        Document updateDocument = new Document("$set", new Document("name", "999"));
        collection.updateMany(query, updateDocument);

        MongoCursor<Document> cursor = collection.find().iterator();

        while (cursor.hasNext()) {
            Document document = cursor.next();
            System.out.println(document.toJson());
        }

        cursor.close();
    }
}

```
![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Mongo_10.png)

Посмотрим измененные данные в БД:

![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Mongo_s2.png)


#### 5.2 Redis
Настраиваем проект IntelliJ IDEA для Redis по аналогии с MongoDB и Добавляем зависимости Redis драйвера в pom.xml файл:
```
    <dependencies>
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>5.1.0</version>
        </dependency>
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>3.7.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-slf4j-impl</artifactId>
            <version>2.23.1</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
```
![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Redis_pom.png)

Подключаем базу данных Redis в IntelliJ IDEA:

![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Redis_BD.png)

Отправляем созданный Java запрос на генерацию 10млн строк и вставку в БД построчно:
```
import redis.clients.jedis.Jedis;
import java.util.UUID;

public class Main {

    public static void main(String[] args) {
        Jedis jedis = new Jedis("redis://192.168.0.91:6379");

        for (int i = 1; i <= 10000000; i++) {
            String id = UUID.randomUUID().toString();
            jedis.hset("hash_key", id, "value" + i);
        }

        jedis.close();
    }
}
```
![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Redis_1.png)

Проверим данные в БД Redis:

![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Redis_2.png)

Отправим запрос конвеерной вставки 10млн записей:
```
import redis.clients.jedis.Jedis;
import redis.clients.jedis.Pipeline;
import java.util.UUID;

public class Main {

    public static void main(String[] args) {
        Jedis jedis = new Jedis("redis://192.168.0.91:6379");
        Pipeline pipeline = jedis.pipelined();

        for (int i = 1; i <= 10000000; i++) {
            String id = UUID.randomUUID().toString();
            pipeline.hset("hash_key", id, "value" + i);
        }

        pipeline.sync();
        jedis.close();
    }
}
```
![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Redis_3.png)

Произведем поиск значений value в которых встречается число 999:
```
import redis.clients.jedis.Jedis;
import redis.clients.jedis.ScanParams;
import redis.clients.jedis.ScanResult;
import java.util.Map;

public class Main {

    public static void main(String[] args) {
        Jedis jedis = new Jedis("redis://192.168.0.91:6379");

        String cursor = "0";
        ScanParams params = new ScanParams().match("*");

        do {
            ScanResult<Map.Entry<String, String>> result = jedis.hscan("hash_key", cursor, params);
            cursor = result.getCursor();

            for (Map.Entry<String, String> entry : result.getResult()) {
                String value = entry.getValue();
                if (value.contains("999")) {
                    System.out.println("ID: " + entry.getKey() + ", Value: " + value);
                }
            }
        } while (!cursor.equals("0"));

        jedis.close();
    }
}
```
![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Redis_4.png)

Поменяем найденные значения с 999 на 777:
```
import redis.clients.jedis.Jedis;
import redis.clients.jedis.ScanParams;
import redis.clients.jedis.ScanResult;
import java.util.Map;

public class Main {

    public static void main(String[] args) {
        Jedis jedis = new Jedis("redis://192.168.0.91:6379");

        String cursor = "0";
        ScanParams params = new ScanParams().match("*");

        do {
            ScanResult<Map.Entry<String, String>> result = jedis.hscan("hash_key", cursor, params);
            cursor = result.getCursor();

            for (Map.Entry<String, String> entry : result.getResult()) {
                String value = entry.getValue();
                if (value.contains("999")) {
                    String newValue = value.replace("999", "777");
                    jedis.hset("hash_key", entry.getKey(), newValue);
                }
            }
        } while (!cursor.equals("0"));

        jedis.close();
    }
}
```
![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Redis_5.png)

Проверим остались ли записи содержащие "999":

![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Redis_6.png)

Выберем записи содержащие "777":

![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Redis_7.png)

Теперь удалим все строки где встречается значение 777:
```
import redis.clients.jedis.Jedis;
import redis.clients.jedis.ScanParams;
import redis.clients.jedis.ScanResult;
import java.util.Map;

public class Main {

    public static void main(String[] args) {
        Jedis jedis = new Jedis("redis://192.168.0.91:6379");

        String cursor = "0";
        ScanParams params = new ScanParams().match("*");

        do {
            ScanResult<Map.Entry<String, String>> result = jedis.hscan("hash_key", cursor, params);
            cursor = result.getCursor();

            for (Map.Entry<String, String> entry : result.getResult()) {
                String value = entry.getValue();
                if (value.contains("777")) {
                    jedis.hdel("hash_key", entry.getKey());
                }
            }
        } while (!cursor.equals("0"));

        jedis.close();
    }
}
```
![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Redis_8.png)
![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Redis_9.png)

Построим временной график отправленных запросов в обе СУБД:

![connect](https://github.com/dushaev7777/NoSQL1989-02-24/blob/main/Image/Project/Result.png)

# 6. Выводы #

Из графика видно, что со вставкой 10млн записей Java запроса лучше справляется Redis, а вот с поиском, обновлением и удалением данных лучше справляется MongoDB.
Кроме того на сервере приложения для массовой вставки пришлось увеличить объем оперативной памяти для проекта MongoDB, это означает 
что при генерации 10млн записей их вес очень ощутим для хранения в оперативной памяти. Соответственно для сервера приложений MongoDB требуется намного больше оперативной памяти нежеле для Redis.

Redis отлично подходит для кэширования и хранения сессионной информации. Например, хранения маршрутов или пользовательских контактов во время онлайн-покупок. Redis подходит для работы с различными счетчиками и метриками. В нем также можно создавать различные стратегии очистки ключей.

MongoDB лучше всего использовать там, где требуется хранить и анализировать большое количество фрагментов информации, не связанных друг с другом:
 * Каталогов интернет-магазинов
 * Системных логов
 * Записей с датчиков и интернета вещей
 * Данных из мобильных приложений
 * Кэша
 * Данных о пользователях сервиса или приложения

Полученные данные помогут с выбором NoSQL решения для определенных Бизнес-процессов.