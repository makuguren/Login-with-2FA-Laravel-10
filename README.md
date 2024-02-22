# Login with 2 Factor Authentication in Laravel 10

Here the Step by Step How to Make Login Authenticator in Laravel 10.

## Step 1: Installing fresh new laravel Application.
Firstly, we are going to install a fresh new laravel application. Run the following command to install a laravel application.

```php
composer create-project laravel/laravel Login-With-2FA-Laravel-10
```

## Step 2: Setting up database configuration
After successfully installing the laravel app then after configuring the database setup. We will open the “.env” file and change the database name, username and password in the env file.

#### .env
```bash
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=Enter_Your_Database_Name
DB_USERNAME=Enter_Your_Database_Username
DB_PASSWORD=Enter_Your_Database_Password
```

## Step 3: Installing Auth Scaffold
Laravel’s laravel/ui package provides a quick way to scaffold all of the routes and views you need for authentication using a few simple commands:

```php
composer require laravel/ui
```

Next, we need to generate auth scaffold with bootstrap, so let’s run the below command:

```
php artisan ui bootstrap --auth
```

Then, install npm packages using the below command:

```
npm install
```

At last, built bootstrap CSS using the below command:

```
npm run build
```

## Step 4: Installing Google Two-Factor Authentication Package
In this step, we will install pragmarx/google2fa-laravel and bacon/bacon-qr-code that way we can use methods of google authentication. so let’s run the below command:

```php
composer require pragmarx/google2fa-laravel
```

```php
composer require bacon/bacon-qr-code
```

Next, we need to publish configuration file, so let’s run the below command:

```php
php artisan vendor:publish --provider="PragmaRX\Google2FALaravel\ServiceProvider"
```
## Step 5: Adding 2FA Middleware
Here, we will add `2FA` middleware in `Kernel.php` file. 2fa middleware added by `google2fa-laravel composer package`. so let’s register it.

#### app/Http/Kernel.php
```php
<?php
  
namespace App\Http;
  
use Illuminate\Foundation\Http\Kernel as HttpKernel;
  
class Kernel extends HttpKernel
{
    ...
    ...
    /**
     * The application's route middleware.
     *
     * These middleware may be assigned to groups or used individually.
     *
     * @var array
     */
    protected $routeMiddleware = [
        '2fa' => \PragmaRX\Google2FALaravel\Middleware::class,
    ];
  
}
```

## Step 6: Creating Migration
Here, we have to create new migration for add `google2fa_secret` column in users table using Laravel php artisan command, so first fire bellow command:

```php
php artisan make:migration add_google_2fa_columns
```
After this command you will find one file in the following path `database/migrations` and you have to put the below code in your migration file.

```php
<?php
  
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;
  
return new class extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->text('google2fa_secret')->nullable();
        });
    }
  
    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
          
    }
};
```

## Step 7: Updating Model
In this step, we will update User model file. so let’s update as the below:

#### app/Models/User.php

```php
<?php
    
namespace App\Models;
  
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;
use Illuminate\Database\Eloquent\Casts\Attribute;
  
class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
 
    /**
     * The attributes that are mass assignable.
     *
     * @var array
     */
    protected $fillable = [
        'name',
        'email',
        'password',
        'google2fa_secret'
    ];
  
    /**
     * The attributes that should be hidden for serialization.
     *
     * @var array
     */
    protected $hidden = [
        'password',
        'remember_token',
    ];
  
    /**
     * The attributes that should be cast.
     *
     * @var array
     */
    protected $casts = [
        'email_verified_at' => 'datetime',
    ];
  
    /** 
     * Interact with the user's first name.
     *
     * @param  string  $value
     * @return \Illuminate\Database\Eloquent\Casts\Attribute
     */
    protected function google2faSecret(): Attribute
    {
        return new Attribute(
            get: fn ($value) =>  decrypt($value),
            set: fn ($value) =>  encrypt($value),
        );
    }
}
```

## Step 8: Creating Routes
In this step, we will create three routes for v complete registration process with auth routes. So, let’s add a new route to that file.

#### routes/web.php

```php
<?php
  
use Illuminate\Support\Facades\Route;
  
use App\Http\Controllers\Auth\RegisterController;
use App\Http\Controllers\HomeController;
  
/*  
|--------------------------------------------------------------------------
| Web Routes
|--------------------------------------------------------------------------
|
| Here is where you can register web routes for your application. These
| routes are loaded by the RouteServiceProvider within a group which
| contains the "web" middleware group. Now create something great!
|
*/
  
Route::get('/', function () {
    return view('welcome');
});
  
Auth::routes();
   
Route::middleware(['2fa'])->group(function () {
   
    Route::get('/home', [HomeController::class, 'index'])->name('home');
    Route::post('/2fa', function () {
        return redirect(route('home'));
    })->name('2fa');
});
  
Route::get('/complete-registration', [RegisterController::class, 'completeRegistration'])->name('complete.registration');
```

## Step 9: Updating Controller File
In the next step, now we have need to update `RegisterController`, So let’s create a controller:

#### app/Http/Controllers/Auth/RegisterController.php

```php
<?php
  
namespace App\Http\Controllers\Auth;
  
use App\Http\Controllers\Controller;
use App\Providers\RouteServiceProvider;
use App\Models\User;
use Illuminate\Foundation\Auth\RegistersUsers;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Validator;
use Illuminate\Http\Request;
  
class RegisterController extends Controller
{
    /*
    |--------------------------------------------------------------------------
    | Register Controller
    |--------------------------------------------------------------------------
    |
    | This controller handles the registration of new users as well as their
    | validation and creation. By default this controller uses a trait to
    | provide this functionality without requiring any additional code.
    |
    */
  
    use RegistersUsers {
        register as registration;
    }
  
    /**
     * Where to redirect users after registration.
     *
     * @var string
     */
    protected $redirectTo = '/home';
  
    /**
     * Create a new controller instance.
     *
     * @return void
     */
    public function __construct()
    {
        $this->middleware('guest');
    }
  
    /**
     * Get a validator for an incoming registration request.
     *
     * @param  array  $data
     * @return \Illuminate\Contracts\Validation\Validator
     */
    protected function validator(array $data)
    {
        return Validator::make($data, [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'string', 'email', 'max:255', 'unique:users'],
            'password' => ['required', 'string', 'min:6', 'confirmed'],
        ]);
    }
  
    /**
     * Create a new user instance after a valid registration.
     *
     * @param  array  $data
     * @return \App\Models\User
     */
    protected function create(array $data)
    {
        return User::create([
            'name' => $data['name'],
            'email' => $data['email'],
            'password' => Hash::make($data['password']),
            'google2fa_secret' => $data['google2fa_secret'],
        ]);
    }
  
    /**
     * Write code on Method
     *
     * @return response()
     */
    public function register(Request $request)
    {
        $this->validator($request->all())->validate();
  
        $google2fa = app('pragmarx.google2fa');
  
        $registration_data = $request->all();
  
        $registration_data["google2fa_secret"] = $google2fa->generateSecretKey();
  
        $request->session()->flash('registration_data', $registration_data);
  
        $QR_Image = $google2fa->getQRCodeInline(
            config('app.name'),
            $registration_data['email'],
            $registration_data['google2fa_secret']
        );
          
        return view('google2fa.register', ['QR_Image' => $QR_Image, 'secret' => $registration_data['google2fa_secret']]);
    }
  
    /**
     * Write code on Method
     *
     * @return response()
     */
    public function completeRegistration(Request $request)
    {        
        $request->merge(session('registration_data'));
  
        return $this->registration($request);
    }
}
```

## Step 10: Creating Blade File
In this step, we need to create three blade file with index.blade.php and register.blade.php files in google2fa, so let’s update following code on it:

#### resources/views/google2fa/index.blade.php

```php
@extends('layouts.app')
  
@section('content')
<div class="container">
    <div class="row justify-content-center align-items-center " style="height: 70vh;S">
        <div class="col-md-8 col-md-offset-2">
            <div class="panel panel-default">
                <div class="panel-heading font-weight-bold">Register</div>
                <hr>
                @if($errors->any())
                    <div class="col-md-12">
                        <div class="alert alert-danger">
                          <strong>{{$errors->first()}}</strong>
                        </div>
                    </div>
                @endif
  
                <div class="panel-body">
                    <form class="form-horizontal" method="POST" action="{{ route('2fa') }}">
                        {{ csrf_field() }}
  
                        <div class="form-group">
                            <p>Please enter the  <strong>OTP</strong> generated on your Authenticator App. <br> Ensure you submit the current one because it refreshes every 30 seconds.</p>
                            <label for="one_time_password" class="col-md-4 control-label">One Time Password</label>
  
                            <div class="col-md-6">
                                <input id="one_time_password" type="number" class="form-control" name="one_time_password" required autofocus>
                            </div>
                        </div>
  
                        <div class="form-group">
                            <div class="col-md-6 col-md-offset-4 mt-3">
                                <button type="submit" class="btn btn-primary">
                                    Login
                                </button>
                            </div>
                        </div>
                    </form>
                </div>
            </div>
        </div>
    </div>
</div>
@endsection
```

#### resources/views/google2fa/register.blade.php

```php
@extends('layouts.app')
  
@section('content')
<div class="container">
    <div class="row">
        <div class="col-md-12 mt-4">
            <div class="card card-default">
                <h4 class="card-heading text-center mt-4">Set up Google Authenticator</h4>
   
                <div class="card-body" style="text-align: center;">
                    <p>Set up your two factor authentication by scanning the barcode below. Alternatively, you can use the code <strong>{{ $secret }}</strong></p>
                    <div>
                        {!! $QR_Image !!}
                    </div>
                    <p>You must set up your Google Authenticator app before continuing. You will be unable to login otherwise</p>
                    <div>
                        <a href="{{ route('complete.registration') }}" class="btn btn-primary">Complete Registration</a>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
@endsection
```
## Step 11: Migrate
In this Step, we need to migrate by typing this command:

```php
php artisan migrate
```

## Step 12: Testing
All Step has been done. Now we are going to test our application. Run the following command in terminal to start the laravel server.

```php
php artisan serve
```

Now open any web browser and open the following link to test the application

```
http://localhost:8000
```
