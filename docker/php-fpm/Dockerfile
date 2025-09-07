FROM node:24-alpine AS node-builder

WORKDIR /var/www

# Copy only package files first to leverage cache
COPY package*.json ./
RUN npm install

# Copy remaining files needed for build
COPY vite.config.js ./
#COPY tsconfig.json ./
COPY resources resources
COPY public public
RUN npm run build

FROM php:8.4-fpm-alpine AS php-stage

# Install system dependencies
RUN apk update && apk add --no-cache \
    icu-dev git bash curl libpng-dev libxml2-dev zip unzip \
    freetype-dev libjpeg-turbo-dev libwebp-dev zlib-dev libzip-dev oniguruma-dev \
    postgresql-dev supervisor autoconf g++ make

# Configure and install PHP extensions
RUN docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-install intl pdo_mysql mbstring exif pcntl bcmath gd zip

# Install Redis extension
RUN pecl install redis && docker-php-ext-enable redis

# Create user
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

# Configure PHP-FPM
RUN sed -i 's/user = www-data/user = appuser/g' /usr/local/etc/php-fpm.d/www.conf && \
    sed -i 's/group = www-data/group = appuser/g' /usr/local/etc/php-fpm.d/www.conf

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
WORKDIR /var/www


# Copy the initialization script
COPY ./docker/php-fpm/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

#set healthcheck
HEALTHCHECK --interval=30s --timeout=10s --retries=3 CMD curl --fail http://localhost/up || exit 1

# Copy composer files first to leverage cache
COPY composer.json composer.lock ./
COPY app/Helpers/helpers.php ./app/Helpers/helpers.php
RUN composer install --no-dev --optimize-autoloader --no-interaction --prefer-dist --no-scripts

# Copy application files
COPY --chown=appuser:appuser . .

# Copy built assets from node-builder
COPY --chown=appuser:appuser --from=node-builder /var/www/public/build /var/www/public/build

# Copy configuration files
#COPY --chown=appuser:appuser docker/supervisord.conf /etc/supervisord.conf
#COPY --chown=appuser:appuser docker/php/php.ini /usr/local/etc/php/conf.d/custom.ini
#COPY --chown=appuser:appuser docker/php/www.conf /usr/local/etc/php-fpm.d/www.conf
RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"
# Copy the initial storage structure
#COPY ./storage /var/www/storage-init

# Create required directories
RUN mkdir -p \
    storage/framework/cache \
    storage/framework/sessions \
    storage/framework/views \
    storage/app/public \
    public/storage \
    bootstrap/cache \
    storage/logs \
    /var/log/supervisor \
    /var/run/php \
    /var/log/php-fpm

# Set permissions
RUN chown -R appuser:appuser \
    storage \
    bootstrap/cache \
    storage/logs \
    /var/log/supervisor \
    /var/run/php \
    /var/log/php-fpm \
    public \
    public/build \
 && chmod -R 775 storage bootstrap/cache public public/build

USER appuser

# Change the default command to run the entrypoint script
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]

# Expose port 9000 and start php-fpm server
EXPOSE 9000
CMD ["php-fpm"]
