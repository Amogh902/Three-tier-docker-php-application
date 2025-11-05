# Three-Tier PHP Registration Form on Docker

This beginner-friendly project demonstrates how to deploy a simple **PHP registration form** using a **three-tier architecture** — all inside Docker containers.

If you’re new to Docker, this is a perfect hands-on example to understand how containers, volumes, networks, and Dockerfiles work together.

---

## Architecture Overview

This app follows a **three-tier architecture**:

| Tier              | Technology  | Purpose                                                            |
| ----------------- | ----------- | ------------------------------------------------------------------ |
| **Web Tier**      | **Nginx**   | Serves the frontend HTML and forwards PHP requests to the app tier |
| **App Tier**      | **PHP-FPM** | Processes PHP logic and interacts with the database                |
| **Database Tier** | **MySQL**   | Stores user data submitted via the form                            |

Each tier runs in its own **container**, and all containers are managed together using **Docker Compose**.

---

## Project Structure

```
threetier/
├── app
│   └── code
│       └── submit.php
├── db
│   ├── Dockerfile
│   └── init.sql
├── docker-compose.yml
└── web
    ├── code
    │   └── signup.html
    └── config
        └── default.conf
```

![](/docker-php-app-img/directory-structure-img.png)

---

## Step-by-Step Setup

### 1️. Install Docker & Docker Compose

If you’re on Linux and using `sudo -i` (root user):

```bash
yum update -y
amazon-linux-extras install docker -y
systemctl start docker
systemctl enable docker

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose --version
```
---

### 2️. Create the Folders

```bash
mkdir -p threetier/{web,app,db}
mkdir -p threetier/web/{code,config}
mkdir -p threetier/app/code
```
---

### 3️. Build the Web Tier (Nginx)

**File:** `web/config/default.conf`

This configuration tells Nginx to send `.php` files to the PHP app container.

```nginx
location ~ \.php$ {
    root           /app;
    fastcgi_pass   app:9000;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME  /app/$fastcgi_script_name;
    include        fastcgi_params;
}
```

**File:** `web/code/signup.html`

A simple registration form:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Signup Form</title>
</head>
<body>
  <h2>Signup Form</h2>
  <form action="submit.php" method="post">
    <label>Name:</label><br>
    <input type="text" name="name" required><br><br>

    <label>Email:</label><br>
    <input type="email" name="email" required><br><br>

    <label>Website:</label><br>
    <input type="url" name="website"><br><br>

    <label>Comment:</label><br>
    <textarea name="comment"></textarea><br><br>

    <label>Gender:</label><br>
    <input type="radio" name="gender" value="female"> Female
    <input type="radio" name="gender" value="male"> Male
    <input type="radio" name="gender" value="other"> Other<br><br>

    <input type="submit" value="Submit">
  </form>
</body>
</html>
```

![](/docker-php-app-img/Blank-signup-img.png)

---

### 4️. Build the App Tier (PHP-FPM)

**File:** `app/code/submit.php`

Handles the form submission and inserts data into the database.

```php
<?php
$name = $_POST['name'];
$email = $_POST['email'];
$website = $_POST['website'];
$comment = $_POST['comment'];
$gender = $_POST['gender'];

$conn = mysqli_connect('db', 'root', 'root', 'studentapp');
if (!$conn) {
    die('Connection failed: ' . mysqli_connect_error());
}

$sql = "INSERT INTO users (name, email, website, comment, gender)
        VALUES ('$name', '$email', '$website', '$comment', '$gender')";

if (mysqli_query($conn, $sql)) {
    echo "<h2>Record created successfully!</h2>";
} else {
    echo "Error: " . mysqli_error($conn);
}

mysqli_close($conn);
?>
```
---

### 5️. Build the Database Tier (MySQL)

**File:** `db/Dockerfile`

```dockerfile
FROM mysql
ENV MYSQL_ROOT_PASSWORD=root
ENV MYSQL_DATABASE=studentapp
COPY init.sql /docker-entrypoint-initdb.d/
EXPOSE 3306
```

**File:** `db/init.sql`

```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(20),
    email VARCHAR(100),
    website VARCHAR(255),
    gender VARCHAR(6),
    comment VARCHAR(100)
);
```
---

### 6️. Docker Compose (Glue Everything Together)

**File:** `docker-compose.yml`

```yaml
services:
  web:
    image: nginx
    ports:
      - "80:80"
    volumes:
      - ./web/code/:/usr/share/nginx/html/
      - ./web/config/:/etc/nginx/conf.d/
    depends_on:
      - app
      - db
    networks:
      - webnet

  app:
    image: bitnami/php-fpm
    volumes:
      - ./app/code/:/app
    depends_on:
      - db
    networks:
      - webnet
      - dbnet

  db:
    build: ./db/
    volumes:
      - myvolume:/var/lib/mysql
    networks:
      - dbnet

networks:
  webnet:
  dbnet:

volumes:
  myvolume:
```

![](/docker-php-app-img/creating-containers-img.png)

---

## Run the Application

Start everything:

```bash
docker-compose up -d
```

Check running containers:

```bash
docker ps
```

![](/docker-php-app-img/running-state-contianers-img.png)

---

## Test the Application

Open your browser:

```
http://<your-ec2-public-ip>/signup.html
```

![](/docker-php-app-img/filled-signup-img.png)

Fill the form → Submit.

![](/docker-php-app-img/Successful-register-img.png)

To verify data in MySQL:

```bash
docker exec -it <db_container_id> mysql -u root -p root studentapp
mysql> SELECT * FROM users;
```

![](/docker-php-app-img/Db-container-img.png)

---

## Docker networks

![](/docker-php-app-img/network-img.png)

## Docker Volumes

![](/docker-php-app-img/volume-img.png)

##  Key Docker Concepts Explained

| Concept            | Explanation                                                                              |
| ------------------ | ---------------------------------------------------------------------------------------- |
| **Volumes**        | Used to persist MySQL data (`myvolume`) and share code between your host and containers. |
| **Networks**       | `webnet` connects Nginx ↔ PHP-FPM, while `dbnet` connects PHP-FPM ↔ MySQL.               |
| **Dockerfile**     | Custom image instructions for the MySQL container, including preloaded schema.           |
| **Docker Compose** | The YAML file that defines and connects all services together.                           |

---


