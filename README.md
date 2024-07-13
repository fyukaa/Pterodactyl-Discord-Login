<!-- Upload files --!>
1. Drag all the given files in the /var/www/pterodactyl directory.


<!-- Downloading required packages --!>
2. Run `composer require laravel/socialite` in /var/www/pterodactyl.


<!-- Updating the Database --!>
3. Run `php artisan migrate` in /var/www/pterodactyl.


<!-- Creating Discord App --!>
4. Please create a new Discord application @ https://discordapp.com/developers/applications and configure the OAuth Redirect URL to : "https://my.panel/auth/login/sso"
(replace my.panel with your panel domain) with the identify and email permissions.


<!-- Setting up .env --!>
5. Open the /var/www/pterodactyl/.env file (this file might be hidden because it has a dot before the name, google: "Your program name: show hidden files".)
   add the following lines on the bottom of your file, and replace the "CLIENT ID" by your "CLIENT ID" generated on the Discord Application. + replace the "CLIENT SECRET" by your "CLIENT SECRET" generated on the Discord Application.


DISCORD_CLIENT_ID=your_client_id
DISCORD_CLIENT_SECRET=your_client_secret


<!-- Editing LoginController --!>
6. Open /app/Http/Controllers/Auth/LoginController.php

/app/Http/Controllers/Auth/LoginController.php
Insert this codes
    
use Illuminate\Support\Str;
use Pterodactyl\Services\Users\UserCreationService;
use Illuminate\Support\Facades\Auth;    
use Illuminate\Support\Facades\Redirect;
use Pterodactyl\Models\User;

Above: 

use Illuminate\Http\Request;




/app/Http/Controllers/Auth/LoginController.php
Insert this codes
    
use Laravel\Socialite\Facades\Socialite;

Under

use Illuminate\Auth\AuthManager;




/app/Http/Controllers/Auth/LoginController.php
Insert this codes

/**
 * @var \App\Services\Users\UserCreationService
 */
protected $creationService;

Under:

protected $maxAttempts;




/app/Http/Controllers/Auth/LoginController.php
Insert this codes

UserCreationService $creationService

Under:

UserRepositoryInterface $repository,




/app/Http/Controllers/Auth/LoginController.php
Insert this codes

* @param UserCreationService $creationService

Under:

* @param \Pterodactyl\Contracts\Repository\UserRepositoryInterface $repository




/app/Http/Controllers/Auth/LoginController.php
Insert this codes

$this->creationService = $creationService;

Under:

$this->repository = $repository;




/app/Http/Controllers/Auth/LoginController.php
Insert this codes

if($request->input("discord-sso")) {
    return $this->redirectToProvider();
}

Above:

$username = $request->input($this->username());




/app/Http/Controllers/Auth/LoginController.php
Insert this codes

public function redirectToProvider()
{
    return Socialite::driver('discord')
        ->setScopes(["identify", "email"])
        ->redirect();
}

public function handleProviderCallback()
{
    try {
        $user = Socialite::driver('discord')->user();
    } catch (Exception $e) {
        return Redirect::to("auth/login");
    }

    $authUser = $this->findOrCreateUser($user);

    Auth::login($authUser, true);

    return Redirect::route('index');
}

/**
 * Return user if exists; create and return if doesn't
 *
 * @param $discordUser
 *
 * @return User
 * @throws \App\Exceptions\Model\DataValidationException
 */

private function findOrCreateUser($discordUser)
{
    if ($authUser = $this->repository->findWhere([['discord_user_token', '=', $discordUser->id]])->first()) {
        return $authUser;
    }
    if($authUser = $this->repository->findWhere([['email', '=', $discordUser->email]])->first()) {
        $authUser->update([
            'discord_user_token' => $discordUser->id
        ]);
        return $authUser;
    }

    return $this->creationService->handle([
        'username' => $discordUser->name,
        'email' => $discordUser->email,
        'name_first' => $discordUser->name,
        'name_last' => $discordUser->name,
        'discord_user_token' => $discordUser->id
    ]);
}

Under:

private function fireFailedLoginEvent(Authenticatable $user = null, array $credentials = [])
{
    event(new Failed(config('auth.defaults.guard'), $user, $credentials));
}



<!-- Editing VerifyReCaptcha --!>
7. Open /app/Http/Middleware/VerifyReCaptcha.php

/app/Http/Middleware/VerifyReCaptcha.php
replace this codes

if (! $this->config->get('recaptcha.enabled') {
    return $next($request);
}

With this:

if (! $this->config->get('recaptcha.enabled') || $request->filled('discord-sso')) {
    return $next($request);
}




<!-- Editing User --!>
8. Open /app/Models/User.php

/app/Models/User.php
Insert this codes

'discord_user_token',

Under:

'root_admin'



<!-- Editing AppServiceProvider --!>
8. Open /app/Providers/AppServiceProvider.php

/app/Providers/AppServiceProvider.php
Insert this codes

use Pterodactyl\Socialite\DiscordProvider;

Under:

use Pterodactyl\Observers\SubuserObserver;




/app/Providers/AppServiceProvider.php
Insert this codes

$socialite = $this->app->make('Laravel\Socialite\Contracts\Factory');
$socialite->extend(
    'discord',
    function ($app) use ($socialite) {
        $config = $app['config']['services.discord'];
        return $socialite->buildProvider(DiscordProvider::class, $config);
    }
);

Under:

Theme::setSetting('cache-version', md5($this->versionData()['version'] ?? 'undefined'));




/app/Providers/AppServiceProvider.php
Replace this codes

return Cache::remember('git-version', 5, function () {

With This:

return Cache::remember('git-version', now()->addMinutes(5), function () {




<!-- Editing app config --!>
8. Open /config/app.php

/config/app.php
Insert this codes

Laravel\Socialite\SocialiteServiceProvider::class,

Under:

Prologue\Alerts\AlertsServiceProvider::class,




/config/app.php
Insert this codes

'Socialite' => Laravel\Socialite\Facades\Socialite::class,

Under:

'Storage' => Illuminate\Support\Facades\Storage::class,



<!-- Editing auth routes --!>
8. Open /routes/auth.php

/routes/auth.php
Insert this codes

Route::get('/login/sso', 'LoginController@handleProviderCallback')->name('auth.login.sso');

Under:

Route::get('/login/totp', 'LoginController@totp')->name('auth.totp');




<!-- Editing services --!>
8. Open /config/services.php

/config/services.php
Insert this codes

'discord' => [
    'client_id' => env('DISCORD_CLIENT_ID'),
    'client_secret' => env('DISCORD_CLIENT_SECRET'),
    'redirect' => env('DISCORD_REDIRECT', env('APP_URL') . '/auth/login/sso'),
],

Under:

'sparkpost' => [
    'secret' => env('SPARKPOST_SECRET'),
],




<!-- Updating the default layout --!>
8. Open the /resources/themes/pterodactyl/auth/login.blade.php

/resources/themes/pterodactyl/auth/login.blade.php
Insert This Codes

<div class="row">
    <h4 class="text-center" style="color: #fff;">OR</h4>
    <div class="col-lg-12 text-center">
        <form role="form" id="loginWithDiscord" action="{{ route('auth.login') }}" method="POST">
            @csrf
            <input type="hidden" name="discord-sso" value="discord-sso">
            <input type="image" src="https://i.ibb.co/HTCDZW8/discord.png" alt="Login With Discord" style="height: 40px; border-radius: 5px;">
        </form>
    </div>
<div>

Under

</form>
