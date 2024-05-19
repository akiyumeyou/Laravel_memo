めもファイルがもしかしたら見えないかもしれないのでここにも貼っておきます。

phpMYadmin http://localhost:8888/
*****************************************
VScodeを使う　Docker laravel環境設定 　メモ（Mac)
*****************************************
1.Dockerをインストールする
　（初回のみ　DockerのページからパソコンのOSに合わせたものをダウンロードしてインストール）
Dockerを起動しておく
開発用フォルダを作成。右クリックしてターミナルを表示。以下にすすむ

# 新しいプロジェクトの作成
curl -s https://laravel.build/example-app | bash
cd example-app
curl -s https://laravel.build/appの名前 | bash
cd appの名前

2.phpMyAdminの追加
docker-compose.yml
最後のnetworks:の直前に下記を追記
※インデントが重要なので勝手に整形しないように
==================================
    phpmyadmin:
        image: phpmyadmin/phpmyadmin
        depends_on:
            - mysql
        ports:
            - 8888:80
        environment:
            PMA_USER: '${DB_USERNAME}'
            PMA_PASSWORD: '${DB_PASSWORD}'
            PMA_HOST: mysql
        networks:
            - sail
==================================

# 既存プロジェクトに Sail をインストール
３ 既存プロジェクトに Sail をインストール（appターミナルから）
composer require laravel/sail --dev

# Sail をインストール
php artisan sail:install
※初回のみ設定のエイリアスの登録は省略

# Docker コンテナの起動
# Docker コンテナの起動しておく
./vendor/bin/sail up
（次からは　sail up のみでOK / 終了は sail down)

環境設定終了
＊＊＊＊＊＊
４.ログインおよびGoogleログインの実装
＊＊＊＊＊＊
# Breeze をインストール
./vendor/bin/sail composer require laravel/breeze --dev
./vendor/bin/sail artisan breeze:install
@@ -25,11 +60,13 @@ php artisan sail:install
# 確認 ユーザーコントローラー
sail artisan make:controller UserController -r --model=User

6. HTML/CSS/JSをビルド（構築👉フロントで修正があるたびにビルド）
# HTML/CSS/JSをビルド（構築👉フロントで修正があるたびにビルド）
npm run build
7．テーブル作成

# テーブル作成
sail artisan migrate
ログイン機能・画面が作成されました、画面で確認！。

基本ログイン機能・画面が作成されました、画面で確認！。

3．Login画面とRegister画面にリンクを追加( 2ファイル修正)
/resources/views/auth/register.blade.php
@@ -49,3 +86,211 @@ php artisan migrate
        @endauth
    </div>
@endif

*********************************************
5.##google認証の実装
Socialiteのインストール
./vendor/bin/sail composer require laravel/socialite

GCコンソールで認証情報登録
Googleの画面→キーを取得し.envファイルの一番下に以下の形で記述

GOOGLE_KEY="******************************"
GOOGLE_SECRET="******************************"
GOOGLE_REDIRECT_URI="http://localhost/auth/google/callback"

＊＊＊＊次に、service/config.appにGoogleとの連携のための設定を記述します

config.php
    'google' => [
        'client_id' => env('GOOGLE_KEY'), 
        'client_secret' => env('GOOGLE_SECRET'), 
        'redirect' => env('GOOGLE_REDIRECT_URI'), 
      ],

＊＊＊＊＊
コントローラー
Googleアカウントを使用してログイン機能を実装するためのコントローラを作成します

./vendor/bin/sail artisan make:migration modify_password_column_nullable_in_users_table --table=users
＊＊＊＊＊＊
GoogleLoginController.phpファイルの中身は以下のように書き換えます

GoogleLoginController.php
<?php

namespace App\Http\Controllers;

use Laravel\Socialite\Facades\Socialite;
use App\Models\User;
use Illuminate\Support\Facades\Auth;
use Exception;
use Illuminate\Support\Facades\Log;

class GoogleLoginController extends Controller
{
    public function redirectToGoogle()
    {
        return Socialite::driver('google')->redirect();
    }

    public function handleGoogleCallback()
    {
        try {
            $socialiteUser = Socialite::driver('google')->user();
            $email = $socialiteUser->email;

            $user = User::firstOrCreate(['email' => $email], [
                'name' => $socialiteUser->name,
            ]);

            Auth::login($user);

            return redirect()->intended('dashboard');
        } catch (Exception $e) {
            Log::error($e);
            throw $e;
        }
    }
}
＊＊＊＊
ルーティング
web.phpにGoogleLoginControllerを追加し、認証用のルートを定義します（全部張り替え）

use App\Http\Controllers\GoogleLoginController;

Route::get('/auth/google', [GoogleLoginController::class, 'redirectToGoogle'])
    ->name('login.google');

Route::get('/auth/google/callback', [GoogleLoginController::class, 'handleGoogleCallback'])
    ->name('login.google.callback');

＊＊＊＊＊
ビュー
ログイン画面のBladeファイルを以下のように変更します
=========================================================
<x-guest-layout>
    @if (Route::has('login'))
        <div class="hidden fixed top-0 right-0 px-6 py-4 sm:block">
            @auth
                <a href="{{ url('/dashboard') }}" class="text-sm text-gray-700 dark:text-gray-500 underline">Dashboard</a>
            @else
                <a href="{{ route('login') }}" class="text-sm text-gray-700 dark:text-gray-500 underline">Log in</a>

                @if (Route::has('register'))
                    <a href="{{ route('register') }}" class="ml-4 text-sm text-gray-700 dark:text-gray-500 underline">Register</a>
                @endif
            @endauth
        </div>
    @endif
    <!-- Session Status -->
    <x-auth-session-status class="mb-4" :status="session('status')" />

    <form method="POST" action="{{ route('login') }}">
        @csrf

        <!-- Email Address -->
        <div>
            <x-input-label for="email" :value="__('Email')" />
            <x-text-input id="email" class="block mt-1 w-full" type="email" name="email" :value="old('email')" required autofocus autocomplete="username" />
            <x-input-error :messages="$errors->get('email')" class="mt-2" />
        </div>

        <!-- Password -->
        <div class="mt-4">
            <x-input-label for="password" :value="__('Password')" />

            <x-text-input id="password" class="block mt-1 w-full"
                            type="password"
                            name="password"
                            required autocomplete="current-password" />

            <x-input-error :messages="$errors->get('password')" class="mt-2" />
        </div>

        <!-- Remember Me -->
        <div class="block mt-4">
            <label for="remember_me" class="inline-flex items-center">
                <input id="remember_me" type="checkbox" class="rounded dark:bg-gray-900 border-gray-300 dark:border-gray-700 text-indigo-600 shadow-sm focus:ring-indigo-500 dark:focus:ring-indigo-600 dark:focus:ring-offset-gray-800" name="remember">
                <span class="ml-2 text-sm text-gray-600 dark:text-gray-400">{{ __('Remember me') }}</span>
            </label>
        </div>

        <div class="flex items-center justify-between mt-4">
            <div>
                @if (Route::has('password.request'))
                    <a class="underline text-sm text-gray-600 dark:text-gray-400 hover:text-gray-900 dark:hover:text-gray-100 rounded-md focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500 dark:focus:ring-offset-gray-800" href="{{ route('password.request') }}">
                        {{ __('Forgot your password?') }}
                    </a>
                @endif
            </div>

            <div>
                <x-primary-button>
                    {{ __('Log in') }}
                </x-primary-button>
            </div>
        </div>
    </form>

    <div class="flex items-center justify-end mt-4">
        <a href="{{ route('login.google') }}" class="ml-3 inline-flex items-center">
            <img src="https://developers.google.com/identity/images/btn_google_signin_dark_normal_web.png" style="margin-left: 3em;">
        </a>
    </div>
</x-guest-layout>
=========================================================
＊＊＊
マイグレーション
Googleログイン時はパスワードは使用しないので、パスワードをNULL許容するようにpasswordカラムを変更いたします
以下コマンドでマイグレーションファイルを生成します

./vendor/bin/sail artisan make:migration modify_password_column_nullable_in_users_table --table=users
＊＊＊
生成されたマイグレーションファイルの中身を以下のように書き換えます
========================================================
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class ModifyPasswordColumnNullableInUsersTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('password')->nullable()->change();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('password')->nullable(false)->change();
        });
    }
}
=============================================================

＊＊＊
sail artisan migrate
***

で無事完了



