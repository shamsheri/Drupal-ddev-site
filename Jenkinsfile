pipeline {
  agent {
    docker {
      image 'php:8.3-cli'
      args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }

  options {
    timestamps()
    ansiColor('xterm')
    skipDefaultCheckout(true)
  }

  parameters {
    booleanParam(name: 'CLEANUP', defaultValue: false, description: 'Cleanup containers & network after build')
  }

  environment {
    NET_NAME       = 'drupal-ci-net'
    DB_CONTAINER   = 'drupal-db-ci'
    WEB_CONTAINER  = 'drupal-web-ci'

    DB_NAME        = 'drupal'
    DB_USER        = 'drupal'
    DB_PASS        = 'drupal'
    DB_ROOT_PASS   = 'root'
    DB_IMAGE       = 'mariadb:10.11'

    // Composer quality-of-life
    COMPOSER_MEMORY_LIMIT = '-1'
    COMPOSER_ALLOW_SUPERUSER = '1'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          userRemoteConfigs: [[url: 'https://github.com/shamsheri/Drupal-ddev-site.git']],
          branches: [[name: '*/main']]
        ])
      }
    }

    stage('Pre-clean (optional)') {
      steps {
        sh '''
          set -e
          docker rm -f ${WEB_CONTAINER} ${DB_CONTAINER} >/dev/null 2>&1 || true
          docker network rm ${NET_NAME} >/dev/null 2>&1 || true
        '''
      }
    }

    stage('Install tools (composer, mariadb-client, docker cli)') {
      steps {
        sh '''
          set -e
          apt-get update
          # mysql-client -> mariadb-client (Debian new name)
          apt-get install -y git unzip curl ca-certificates mariadb-client docker.io tar
          curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
          php -v
          composer -V
        '''
      }
    }

    stage('Create network & start DB') {
      steps {
        sh '''
          set -e
          docker network create ${NET_NAME} || true
          docker rm -f ${DB_CONTAINER} >/dev/null 2>&1 || true

          docker run -d --name ${DB_CONTAINER} \
            --network ${NET_NAME} \
            -e MYSQL_DATABASE=${DB_NAME} \
            -e MYSQL_USER=${DB_USER} \
            -e MYSQL_PASSWORD=${DB_PASS} \
            -e MYSQL_ROOT_PASSWORD=${DB_ROOT_PASS} \
            ${DB_IMAGE} \
            --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci

          echo "Waiting for DB to be ready..."
          for i in $(seq 1 60); do
            if docker exec ${DB_CONTAINER} sh -lc "mysqladmin ping -h 127.0.0.1 -uroot -p${DB_ROOT_PASS} --silent"; then
              echo "DB is up"; break
            fi
            sleep 2
          done
        '''
      }
    }
stage('Composer install') {
  steps {
    sh '''
      set -e
      # Fix Git safe directory issue
      git config --global --add safe.directory /var/lib/jenkins/workspace/drupal-pipeline

      # Install required PHP extensions for Drupal
      apt-get update
      apt-get install -y libpng-dev libjpeg-dev libfreetype6-dev
      docker-php-ext-configure gd --with-freetype --with-jpeg
      docker-php-ext-install gd

      export COMPOSER_MEMORY_LIMIT=${COMPOSER_MEMORY_LIMIT}
      export COMPOSER_ALLOW_SUPERUSER=${COMPOSER_ALLOW_SUPERUSER}
      composer install --no-interaction --prefer-dist --no-progress
    '''
  }
}
    stage('Prepare files & settings.php') {
      steps {
        sh '''
          set -e
          mkdir -p web/sites/default/files
          chmod -R 777 web/sites/default/files || true

          if [ -f files.tar.gz ]; then
            mkdir -p web/sites/default
            # handle both structures: files/ or its inner contents
            if tar -tzf files.tar.gz | head -1 | grep -q '^files/'; then
              tar -xzf files.tar.gz -C web/sites/default
            else
              mkdir -p web/sites/default/files
              tar -xzf files.tar.gz -C web/sites/default/files
            fi
          fi

          cat > web/sites/default/settings.php <<'PHP'
<?php
$databases['default']['default'] = [
  'database' => getenv('DB_NAME') ?: 'drupal',
  'username' => getenv('DB_USER') ?: 'drupal',
  'password' => getenv('DB_PASS') ?: 'drupal',
  'host'     => 'drupal-db-ci',
  'port'     => '3306',
  'driver'   => 'mysql',
  'prefix'   => '',
  'collation'=> 'utf8mb4_general_ci',
];
$settings['hash_salt'] = 'jenkins-ci-salt';
$settings['config_sync_directory'] = 'config/sync';
$settings['trusted_host_patterns'] = ['.*'];
PHP
        '''
      }
    }

    stage('Import database') {
    steps {
        sh '''
        set -e
        if [ -f drupal_dump.sql ]; then
            echo "Importing drupal_dump.sql ..."
            docker cp drupal_dump.sql ${DB_CONTAINER}:/tmp/dump.sql
            docker exec ${DB_CONTAINER} sh -lc "mysql --binary-mode=1 -u root -p${DB_ROOT_PASS} ${DB_NAME} < /tmp/dump.sql"
        else
            echo "No drupal_dump.sql found - skipping import"
        fi
        '''
    }
}
        stage('Start Web Container') {
      steps {
        sh '''
          set -e
          docker rm -f ${WEB_CONTAINER} >/dev/null 2>&1 || true

          docker run -d --name ${WEB_CONTAINER} \
            --network ${NET_NAME} \
            -p 8081:80 \
            -v $(pwd):/var/www/html \
            php:8.3-apache

          echo "Waiting for Web container to start..."
          sleep 5
        '''
      }
    }
stage('Run Web (Apache + PHP 8.3 + PHP extensions)') {
    steps {
        sh '''
        # Install PHP extensions required by Drupal inside web container
        docker exec ${WEB_CONTAINER} bash -lc '
          set -e
          apt-get update
          apt-get install -y libpng-dev libjpeg-dev libfreetype6-dev \
                             libzip-dev libonig-dev libxml2-dev unzip
          
          # Configure and install PHP extensions
          docker-php-ext-configure gd --with-freetype --with-jpeg
          docker-php-ext-install -j"$(nproc)" gd pdo_mysql opcache zip mbstring
          
          # Enable Apache modules
          a2enmod rewrite headers expires
          
          # Update DocumentRoot to /web for Drupal
          sed -ri -e "s!DocumentRoot /var/www/html!DocumentRoot /var/www/html/web!g" \
              /etc/apache2/sites-available/000-default.conf
          
          # Allow .htaccess overrides in web directory
          sed -ri -e "s!</VirtualHost>!<Directory /var/www/html/web>AllowOverride All Require all granted</Directory>\\n</VirtualHost>!g" \
              /etc/apache2/sites-available/000-default.conf
          
          # Restart Apache
          service apache2 restart
        '
        '''
    }
}

    stage('Verify with Drush') {
      steps {
        sh '''
          set +e
          # Only run if drush exists
          if [ -f vendor/bin/drush ]; then
            docker exec ${WEB_CONTAINER} bash -lc 'php /var/www/html/vendor/bin/drush status || true'
            docker exec ${WEB_CONTAINER} bash -lc 'php /var/www/html/vendor/bin/drush cr -y || true'
          else
            echo "Drush not found in vendor/bin/drush - skipping"
          fi
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Build complete. Open http://localhost:8081"
    }
    always {
      script {
        if (params.CLEANUP) {
          sh 'docker rm -f ${WEB_CONTAINER} ${DB_CONTAINER} || true'
          sh 'docker network rm ${NET_NAME} || true'
        }
      }
    }
  }
}
