Step 1: Install Laravel

first of all we need to get fresh Laravel 8 version application using bellow command, So open your terminal OR command prompt and run bellow command:

composer create-project --prefer-dist laravel/laravel blog

Step 2: Add New Column to Users Table

here, we will create new migration for adding "last_seen" column:

php artisan make:migration add_new_column_last_seen

database/migrations/2021_07_12_032305_add_new_column_last_seen.php

<?php

  

use Illuminate\Database\Migrations\Migration;

use Illuminate\Database\Schema\Blueprint;

use Illuminate\Support\Facades\Schema;

  

class AddNewColumnLastSeen extends Migration

{

    /**

     * Run the migrations.

     *

     * @return void

     */

    public function up()

    {

        Schema::table('users', function(Blueprint $table){

            $table->timestamp('last_seen')->nullable();

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

}

now let's run migration command:

php artisan migrate

now, just add last_seen column on user model as like bellow:

app/Models/User.php

<?php

  

namespace App\Models;

  

use Illuminate\Contracts\Auth\MustVerifyEmail;

use Illuminate\Database\Eloquent\Factories\HasFactory;

use Illuminate\Foundation\Auth\User as Authenticatable;

use Illuminate\Notifications\Notifiable;

  

class User extends Authenticatable

{

    use HasFactory, Notifiable;

    /**

     * The attributes that are mass assignable.

     *

     * @var array

     */

    protected $fillable = [

        'name', 'email', 'password', 'last_seen'

    ];

  

    /**

     * The attributes that should be hidden for arrays.

     *

     * @var array

     */

    protected $hidden = [

        'password', 'remember_token',

    ];

  

    /**

     * The attributes that should be cast to native types.

     *

     * @var array

     */

    protected $casts = [

        'email_verified_at' => 'datetime',

    ];

}

Read Also: Laravel 8 Queue Step by Step Tutorial Example

Step 3: Generate Auth Scaffolding

here, we will use laravel ui for generating auth scaffolding, so let's run bellow command:

php artisan breeze:install
 
php artisan migrate
npm install
npm run dev

Step 4: Create Middleware

here, we will create UserActivity for update last seen time and add online status, let's run bellow command:

php artisan make:middleware UserActivity

now, update middleware code as bellow:

app/Http/Middleware/UserActivity.php

<?php

  

namespace App\Http\Middleware;

  

use Closure;

use Illuminate\Http\Request;

use Auth;

use Cache;

use App\Models\User;

  

class UserActivity

{

    /**

     * Handle an incoming request.

     *

     * @param  \Illuminate\Http\Request  $request

     * @param  \Closure  $next

     * @return mixed

     */

    public function handle(Request $request, Closure $next)

    {

        if (Auth::check()) {

            $expiresAt = now()->addMinutes(2); /* keep online for 2 min */

            Cache::put('user-is-online-' . Auth::user()->id, true, $expiresAt);

  

            /* last seen */

            User::where('id', Auth::user()->id)->update(['last_seen' => now()]);

        }

  

        return $next($request);

    }

}

now register, this middleware to kernel file:

app/Http/Kernel.php

<?php

  

namespace App\Http;

  

use Illuminate\Foundation\Http\Kernel as HttpKernel;

  

class Kernel extends HttpKernel

{

    ...........

    protected $middlewareGroups = [

        'web' => [

            \App\Http\Middleware\EncryptCookies::class,

            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,

            \Illuminate\Session\Middleware\StartSession::class,

            \Illuminate\View\Middleware\ShareErrorsFromSession::class,

            \App\Http\Middleware\VerifyCsrfToken::class,

            \Illuminate\Routing\Middleware\SubstituteBindings::class,

            \App\Http\Middleware\RestrictIpAddressMiddleware::class,

            \App\Http\Middleware\UserActivity::class,

        ],

  

        'api' => [

            'throttle:api',

            \Illuminate\Routing\Middleware\SubstituteBindings::class,

        ],

    ];

    ...........

}

Step 5: Create Route

In this is step we need to create some routes for display online users function.

routes/web.php

<?php

  

use Illuminate\Support\Facades\Route;

  

use App\Http\Controllers\UserController;

  

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

  

Route::get('/home', [App\Http\Controllers\HomeController::class, 'index'])->name('home');

    

Route::get('online-user', [UserController::class, 'index']);

Step 6: Create Controller

in this step, we need to create UserController and add following code on that file:

app/Http/Controllers/UserController.php

<?php

  

namespace App\Http\Controllers;

  

use Illuminate\Http\Request;

use App\Models\User;

  

class UserController extends Controller

{

    /**

     * Display a listing of the resource.

     *

     * @return \Illuminate\Http\Response

     */

    public function index(Request $request)

    {

        $users = User::select("*")

                        ->whereNotNull('last_seen')

                        ->orderBy('last_seen', 'DESC')

                        ->paginate(10);

          

        return view('users', compact('users'));

    }

}

Step 7: Create Blade Files

here, we need to create blade files for users. so let's create one by one files:

resources/views/users.blade.php

@extends('layouts.app')

  

@section('content')

<div class="container">

    <table class="table table-bordered data-table">

        <thead>

            <tr>

                <th>No</th>

                <th>Name</th>

                <th>Email</th>

                <th>Last Seen</th>

                <th>Status</th>

            </tr>

        </thead>

        <tbody>

            @foreach($users as $user)

                <tr>

                    <td>{{ $user->id }}</td>

                    <td>{{ $user->name }}</td>

                    <td>{{ $user->email }}</td>

                    <td>

                        {{ Carbon\Carbon::parse($user->last_seen)->diffForHumans() }}

                    </td>

                    <td>

                        @if(Cache::has('user-is-online-' . $user->id))

                            <span class="text-success">Online</span>

                        @else

                            <span class="text-secondary">Offline</span>

                        @endif

                    </td>

                </tr>

            @endforeach

        </tbody>

    </table>

</div>

@endsection

Now we are ready to run our example and login with user. so run bellow command so quick run:

php artisan serve
