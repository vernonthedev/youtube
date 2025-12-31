# Laravel Supabase | Configuration
---

### Step 1: Create the Schema in Supabase

1. Go to your Supabase **SQL Editor**.
2. Run the following command to create a schema named `laravel` (you can name it whatever you like):

```sql
CREATE SCHEMA IF NOT EXISTS laravel;
```

**Grant Permissions:** Since Supabase uses specific roles for its APIs, you need to ensure the `postgres` role (which Laravel uses) has full access:

```sql
GRANT ALL ON SCHEMA laravel TO postgres;
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA laravel GRANT ALL ON TABLES TO postgres;
```

### Step 2: Configure Laravel to use the Schema

You now need to tell Laravel that it should look into this new schema for its internal business.

### 1. Update your Database Config

Open `config/database.php` and find the `pgsql` section. You will add the `search_path` and `schema` keys:

```php
'pgsql' => [
    'driver' => 'pgsql',
		// THOSE TO BE ADDED.
    'schema' => 'laravel',           // This tells Laravel the "primary" schema for migrations
    'search_path' => 'public,laravel', // This allows Laravel to find tables in both places
    'sslmode' => 'prefer',
],
```

### 2. Move Migration History

To keep the `public` schema completely clean, tell Laravel to store its migration history log in the `laravel` schema. Update the `migrations` key at the bottom of `config/database.php`:

```php
'migrations' => [
    'table' => 'laravel.migrations', // Prefix with your schema name
    'update_date_on_publish' => true,
],
```

### Step 3: Organizing Your Tables

Now that the plumbing is set up, you have to decide where your specific tables go.

- **Option A: Automatic (Everything in `laravel`)**
If you leave `'schema' => 'laravel'` in your config, any new migration you run (`php artisan migrate`) will automatically create tables inside the `laravel` schema.
- **Option B: Hybrid (The Professional Choice)**
Keep your "App Data" (Users, Posts, Orders) in `public` so Supabase's Auto-API can see them, but keep "Laravel Data" (Failed Jobs, Cache, Sessions) in `laravel`.

To do this, specify the schema in your migration files:

```php
**// database/migrations/xxxx_create_jobs_table.php
public function up()
{
    Schema::create('laravel.jobs', function (Blueprint $table) {
        $table->id();
        // ...
    });
}**
```

### 2. Update Service Configs (Crucial!)

Even though your migrations create tables like `laravel.sessions`, Laravel's Session and Cache drivers don't automatically know to look inside the `laravel` schema. You must update their specific config files:

- **`config/session.php`**: Change `'table' => 'sessions'` to `'table' => 'laravel.sessions'`.
- **`config/cache.php`**: Change `'table' => 'cache'` to `'table' => 'laravel.cache'`.
- **`config/queue.php`**: Change `'table' => 'jobs'` to `'table' => 'laravel.jobs'` and `'failed' => ['table' => 'failed_jobs']` to `'failed' => ['table' => 'laravel.failed_jobs']`.

### 3. Be Explicit with the `users` Table

In your migration, you have `Schema::create('users', ...)`. Because your `search_path` starts with `public`, Laravel *should* put it in public. However, for total clarity, change it to:

PHP

```php
Schema::create('public.users', function (Blueprint $table) {
   ...
});
```

This ensures that no matter what your config says, your "Living Room" data always stays in the `public` schema.

### 4. Environment Configuration

```bash
DB_CONNECTION=pgsql
DB_HOST=aws-1-eu-west-1.pooler.supabase.com
DB_PORT=6543
DB_DATABASE=postgres
DB_USERNAME=postgres.[project-id]
DB_PASSWORD=[password]
```

### 4. Test the Connection

Once you’ve saved your `.env` and config, run the following command in your terminal to see if Laravel can talk to Supabase:

```bash
php artisan db:monitor
```

### 5. Run Your Migrations

Now you can push your local table structure to the Supabase cloud:

```bash
php artisan migrate
```


```bash
php artisan migrate
```
