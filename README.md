<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/f7979e5f-b821-4a06-9b1c-11f0a8e2c399" />
[Claude link](https://claude.ai/share/70cbf301-3488-441a-9951-5b8a92b600b4)

# Laravel + Breeze + Filament + Spatie Permission + Filament Shield

Full Role-Based Access Control (RBAC) setup for a Laravel admin panel using Filament, with Spatie Permission as the underlying permission engine and Filament Shield as the bridge that connects Filament panels to Spatie Permission.

## Tech Stack

| Package | Version | Purpose |
|---|---|---|
| PHP | ^8.2 \| ^8.3 | Runtime |
| Laravel | ^12.0 | Framework |
| Laravel Breeze | latest | Authentication scaffolding |
| Filament | ^3.0-stable | Admin panel builder |
| spatie/laravel-permission | ^6.0 | Role & permission engine |
| bezhansalleh/filament-shield | ^3.0 | Auto-generates permissions & manages Roles UI inside Filament |

> **Note:** Filament Shield 3.x only supports `spatie/laravel-permission` ^5.0 or ^6.0. If Composer pulls in a newer `spatie/laravel-permission` version by default, you must pin it to `^6.0` (see Troubleshooting below).

---

## Prerequisites

- PHP >= 8.2
- Composer >= 2.x
- MySQL (or any Laravel-supported DB)
- Node.js + npm (for frontend assets)

Check versions:

```bash
php --version
composer --version
```

---

## Step 1 — Create Laravel Project

```bash
composer create-project laravel/laravel . "^12.0"
```

Configure your database in `.env`:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel_project
DB_USERNAME=root
DB_PASSWORD=
```

---

## Step 2 — Install Laravel Breeze

```bash
composer require laravel/breeze --dev
php artisan breeze:install
```

When prompted, choose the **Blade** stack (simplest option — Filament will serve as the admin panel, Breeze only handles the public-facing auth).

Then install frontend deps and run migrations:

```bash
npm install
npm run build
php artisan migrate
```

---

## Step 3 — Install Spatie Laravel-Permission

```bash
composer require spatie/laravel-permission:"^6.0"
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
php artisan migrate
```

Add the `HasRoles` trait to `app/Models/User.php`:

```php
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable
{
    use HasRoles;
    // ...
}
```

---

## Step 4 — Install Filament

```bash
composer require filament/filament:"^3.0-stable" -W
php artisan filament:install --panels
```

Create your first admin user:

```bash
php artisan make:filament-user
```

Start the server and verify the panel loads:

```bash
php artisan serve
```

Visit: `http://localhost:8000/admin`

---

## Step 5 — Install Filament Shield

```bash
composer require bezhansalleh/filament-shield:"^3.0" -W
```

> If Composer throws a version conflict on `spatie/laravel-permission`, run this instead so both packages resolve to compatible versions together:
> ```bash
> composer require spatie/laravel-permission:"^6.0" bezhansalleh/filament-shield:"^3.0" -W
> ```

Install Shield:

```bash
php artisan shield:install
```

- Select your panel (usually `admin`) when prompted.
- Shield automatically registers `FilamentShieldPlugin` in your `AdminPanelProvider.php`. If it doesn't, add it manually:

```php
use BezhanSalleh\FilamentShield\FilamentShieldPlugin;

->plugin(FilamentShieldPlugin::make())
```

### Create a Super Admin

```bash
php artisan shield:super-admin
```

Pick a user from the table shown — that user gets the `super_admin` role with full access to everything, including future resources.

---

## Step 6 — Create a Filament Resource + Generate Permissions

Example with a `Post` model:

```bash
php artisan make:model Post -m
php artisan migrate
php artisan make:filament-resource Post --generate
```

Generate Shield permissions for this resource:

```bash
php artisan shield:generate --resource=PostResource
```

Or generate permissions for **all** existing resources at once:

```bash
php artisan shield:generate --all
```

New permissions (`view_post`, `create_post`, `update_post`, `delete_post`, etc.) now appear under **Filament Shield → Roles** in the admin panel, ready to attach to any role.

---

## Step 7 — User Resource with Role Management

To manage users and assign roles directly from the admin panel, generate a User resource:

```bash
php artisan make:filament-resource User --generate
```

Edit `app/Filament/Resources/UserResource.php`:

```php
use Filament\Forms\Components\CheckboxList;
use Illuminate\Support\Facades\Hash;

public static function form(Form $form): Form
{
    return $form
        ->schema([
            Forms\Components\TextInput::make('name')
                ->required()
                ->maxLength(255),
            Forms\Components\TextInput::make('email')
                ->email()
                ->required()
                ->maxLength(255),
            Forms\Components\DateTimePicker::make('email_verified_at'),
            Forms\Components\TextInput::make('password')
                ->password()
                ->dehydrated(fn ($state) => filled($state))
                ->dehydrateStateUsing(fn ($state) => Hash::make($state))
                ->required(fn (string $context): bool => $context === 'create')
                ->maxLength(255),

            CheckboxList::make('roles')
                ->relationship('roles', 'name')
                ->columns(2)
                ->columnSpanFull(),
        ]);
}

public static function table(Table $table): Table
{
    return $table
        ->columns([
            Tables\Columns\TextColumn::make('name')->searchable(),
            Tables\Columns\TextColumn::make('email')->searchable(),
            Tables\Columns\TextColumn::make('roles.name')
                ->badge()
                ->separator(',')
                ->label('Roles'),
            Tables\Columns\TextColumn::make('email_verified_at')
                ->dateTime()
                ->sortable(),
            Tables\Columns\TextColumn::make('created_at')
                ->dateTime()
                ->sortable()
                ->toggleable(isToggledHiddenByDefault: true),
            Tables\Columns\TextColumn::make('updated_at')
                ->dateTime()
                ->sortable()
                ->toggleable(isToggledHiddenByDefault: true),
        ])
        ->filters([])
        ->actions([
            Tables\Actions\EditAction::make(),
        ])
        ->bulkActions([
            Tables\Actions\BulkActionGroup::make([
                Tables\Actions\DeleteBulkAction::make(),
            ]),
        ]);
}
```

This adds:
- A **CheckboxList** on the User form to assign one or more roles (via Spatie's `roles()` relationship — no extra controller code needed).
- Safe password handling: leaving the password field empty on **Edit** keeps the existing hashed password unchanged.
- A **Roles** badge column on the Users table for quick visibility.

---

## Creating New Roles

1. Go to **Filament Shield → Roles → Create Role** in the admin panel.
2. Name the role (e.g. `editor`, `manager`).
3. Tick the permissions this role should have (auto-listed per resource, e.g. `view_post`, `create_post`).
4. Save.
5. Go to **Users**, edit any user, tick the new role in the Roles checkbox list, and save.

---

## Troubleshooting

### Composer conflict: `spatie/laravel-permission` version mismatch

**Error:**
```
- bezhansalleh/filament-shield ... requires spatie/laravel-permission ^5.0|^6.0
  -> found spatie/laravel-permission[...] but it conflicts with your root composer.json require (^8.3)
```

**Cause:** Filament Shield 3.x only supports Spatie Permission ^5 or ^6. If your `composer.json` already pins Spatie Permission to a newer major version (e.g. ^8.3), Composer won't downgrade it automatically even with `-W`, because `-W` only affects **locked** versions, not explicit constraints in `composer.json`.

**Fix:** Require both packages together with an explicit version so the constraint in `composer.json` gets rewritten:

```bash
composer require spatie/laravel-permission:"^6.0" bezhansalleh/filament-shield:"^3.0" -W
```

Or manually edit `composer.json`:
```json
"spatie/laravel-permission": "^6.0",
```
then run:
```bash
composer update
```

---

## Project Structure Reference

```
app/
  Filament/
    Resources/
      PostResource.php
      UserResource.php
      UserResource/Pages/
        ListUsers.php
        CreateUser.php
        EditUser.php
  Models/
    User.php          <- uses HasRoles trait
    Post.php
  Providers/
    Filament/
      AdminPanelProvider.php   <- FilamentShieldPlugin registered here
```

---

## Useful Commands Cheat Sheet

```bash
# Create a new Filament resource with permissions
php artisan make:filament-resource ModelName --generate
php artisan shield:generate --resource=ModelNameResource

# Generate permissions for ALL existing resources
php artisan shield:generate --all

# Assign super admin
php artisan shield:super-admin

# Create a new admin/panel user manually
php artisan make:filament-user
```

---

## License

This project is open-sourced for personal/educational use.
