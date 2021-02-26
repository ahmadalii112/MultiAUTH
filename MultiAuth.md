**Multi Auth**
<br><br>

**_1. Install Laravel 8_**
<br>
`laravel new MultiAuth`
<br>
`php artsian serve`
<br>
----------------------------------------------------------
**_2. Install Laravel 8 Auth **JETSTREAM**_**
<br>
`composer require laravel/jetstream`
<br>
`php artisan jetstream:install install livewire`
<br>
`npm install && npm run dev`
----------------------------------------------------------
**_3. Migration_**
<br>
`php artsian migrate`
<br>
Register and Login
----------------------------------------------------------
**_4. Deafult Auth System Profile Update_**

a. **config->fortify.php** => features Add or Remove Features

b. **config=>jetstream.php** => features Add or Remove Features

c. uncomment ProfilePhoto in **jetstream.php**

d. **.env File**

APP_URL paste your SiteURL e.g http://127.0.0.1:8000/

`php artisan config:cache`
<br>
`php artisan storage:link`

Add Profile Photo Again And it works

----------------------------------------------------------
**_5. Setup Admin Table and Seed data_**

`php artisan make:controller AdminController`
<br>
`php artisan make:model Admin -m`
<br>
Copy all User Model & Migration table to Admin Model & Migration table

Make Factory
<br>
`php artisan make:factory AdminFactory`
<br>
Copy userfactory return to adminfactory and add data

`
return [

    'name' => 'Admin',
    'email' => 'admin@gmail.com',
    'email_verified_at' => now(),
    'password' => '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', // password
    'remember_token' => Str::random(10),
];
`

Make Seeder
<br>
`php artisan make:seeder UserSeeder`

In the DatabaseSeeder

`\App\Models\Admin::factory()->create();`

Migrate Seed<br>
`php artisan migrate --seed`

----------------------------------------------------------
_**6. Create Guards for admin**_

### **GUARD**

Config=> Auth

'guards' => [

    'admin' => [
    'driver' => 'session',
    'provider' => 'admins',
],
]
-----------------------
Also add providers of same name as I do here 

'providers' => [

    'admins' => [
    'driver' => 'eloquent',
    'model' => App\Models\Admin::class,
    ],
]

-----------------------
Also add passwords like given below

'passwords' => [

    'admins' => [
    'provider' => 'admins',
    'table' => 'password_resets',
    'expire' => 60,
    'throttle' => 60,
    ],

]

In the providers

**FortifyServiceProvider.php**

public function register()
{
 
    $this->app->when([
            AdminController::class,
            AttemptToAuthenticate::class,
            RedirectIfTwoFactorAuthenticatable::class
        ])->needs(StatefulGuard::class)->give(function(){
            return Auth::guard('admin');
        });
}

Copy all StatefulGuard.php

"vendor\laravel\framework\src\Illuminate\Contracts\Auth\StatefulGuard.php"

Make New Folder in app name **Guards** 
create a file **AdminStatefulGuard**.php
and paste it<br>
use this <br>
`namespace App\Guards;`
<br>
`use Illuminate\Contracts\Auth\Guard;`

----------------------------------------------------------

**_7. Laravel 8 Multi Auth Part 1_**

"Laravel/fortify/src/http/controllers/AuthenticatedSessionController"<br>
copy all of it and paste on<br>
"`AdminController.php`"
use this on top 
`namespace App\Http\Controllers;`

### routes/web.php


    Route::get('/', function () {
    return view('welcome');
    });

    Route::group(['prefix'=>'admin','middleware'=>['admin:admin']],function (){
    Route::get('/login',[AdminController::class,'loginForm']);
    Route::post('/login',[AdminController::class,'store'])->name('admin.login');
    });
    
    #Admin
    Route::middleware(['auth:sanctum,admin', 'verified'])->get('/admin/dashboard', function () {
    return view('dashboard');
    })->name('dashboard');
    
    
    #Default
    Route::middleware(['auth:sanctum,web', 'verified'])->get('/dashboard', function () {
    return view('dashboard');
    })->name('dashboard');
    



In The "**AdminController.php**"

create a function after constructor

    public function loginForm()
    {
    return view('auth.login',['guard'=>'admin']);
    }
In the View/auth/login.blade.php

    <form method="POST" action="{{ isset($guard) ? url($guard.'/login') : route('login') }}">

Next Step is to Support all the guards of AdminController so <br>
we copy from

    vendor\laravel\fortify\src\Actions\AttemptToAuthenticate.php
    vendor\laravel\fortify\src\Actions\RedirectIfTwoFactorAuthenticatable.php
and Paste on
    
    \app\Actions\Fortify

also change namespace on top of the file

    namespace App\Actions\Fortify;

----------------------------------------------------------
**_8. Laravel 8 Multi Auth Part 2_**

In Providers `RouteServiceprovider.php` create a function
    
    public static function redirectTo($guard)
    {
    return $guard.'/dashboard';
    }

In Middleware `RedirectIfAuthenticated`

Change this to

    public function handle(Request $request, Closure $next, ...$guards)
    {
    $guards = empty($guards) ? [null] : $guards;
        foreach ($guards as $guard) 
        {
            if (Auth::guard($guard)->check()) 
            {
                return redirect(RouteServiceProvider::HOME);
            }
        }
        return $next($request);
    }
This and also ignore 

    public function handle(Request $request, Closure $next, ...$guards)
    {
    $guards = empty($guards) ? [null] : $guards;
        foreach ($guards as $guard) 
        {
            if (Auth::guard($guard)->check()) 
            {
                return redirect($guard.'/dashboard');
            }
        }
        return $next($request);
    }

Copy file App/http/middleware `RedirectifAuthenticated.php` 
and paste it with new name as `AdminRedirectifAuthenticated.php`

Now in `Kernal.php`

    'admin' => \App\Http\Middleware\AdminRedirectIfAuthenticated::class,




if the user logged in then this file redirect

`\vendor\laravel\fortify\src\Http\Responses\LoginResponse.php`

So we create this file for Admin 
create folder in App/http `Responses/LoginResponse.php`


In the App/http `Responses/LoginResponse.php` change this function

    public function toResponse($request)
    {
        return $request->wantsJson()
            ? response()->json(['two_factor' => false])
            : redirect()->intended('/admin/dashboard');
    }


Clear the config cache `php artisan config:clear` or rebuild it `php artisan config:cache`

----------------------------------------------------------
_**9. Laravel 8 Multi Auth Part 3**_

we face Eroor with admin login it not go to the dashboard so we change this
In admin Controller
        
    use App\Http\Responses\LoginResponse;
    //use Laravel\Fortify\Contracts\LoginResponse;

comment the Default Login Response and use our Response


also in   `app/provders/FortifyServiceProvider.php`

    //use Laravel\Fortify\Actions\AttemptToAuthenticate;
    use App\Actions\Fortify\AttemptToAuthenticate;
    use App\Actions\Fortify\RedirectIfTwoFactorAuthenticatable;
    //use Laravel\Fortify\Actions\RedirectIfTwoFactorAuthenticatable;
----------------------------------------------------------
