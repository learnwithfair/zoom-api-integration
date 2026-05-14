
# Zoom Meeting API Integration Documentation (Laravel)

[![Youtube][youtube-shield]][youtube-url]
[![Facebook][facebook-shield]][facebook-url]
[![Instagram][instagram-shield]][instagram-url]
[![LinkedIn][linkedin-shield]][linkedin-url] 

Thanks for visiting my GitHub account!

## Overview

This documentation explains the complete professional process of integrating Zoom Meeting API into a Laravel application using Server-to-Server OAuth.

This guide includes:

* Zoom account setup
* Zoom app creation
* Getting credentials
* Laravel backend integration
* Migration
* Model
* Request validation
* Service layer
* API routes
* Controller
* Postman examples
* Sample responses
* Security best practices
* Professional architecture guidelines

---

# 1. Create Zoom Account

Open: [Zoom Official Website](https://zoom.us/)

Steps:
1. Click `Sign Up`
2. Create your Zoom account
3. Verify your email
4. Login successfully

After login, you will enter Zoom dashboard.

---

# 2. Open Zoom Developer Portal

Open: [Zoom Developer Portal](https://developers.zoom.us/)

Or directly: [Zoom Marketplace Build App](https://marketplace.zoom.us/develop/create)

---

# 3. Create Zoom App

Inside developer portal:

1. Click `Develop`
2. Click `Build App`

You will see several app types.

Choose:

```text
Server-to-Server OAuth
```

Click:

```text
Create
```

---

# 4. Configure App Information

Fill the following fields:

| Field                   | Example                                       |
| ----------------------- | --------------------------------------------- |
| App Name                | Laravel Zoom Integration                      |
| Company Name            | Your Company                                  |
| Developer Contact Email | [admin@example.com](mailto:admin@example.com) |

Click:

```text
Continue
```

---

# 5. Get Zoom Credentials

After app creation you will land on:

```text
App Credentials
```

Here you will get:

```text
Account ID
Client ID
Client Secret
```

Example:

```env
ZOOM_ACCOUNT_ID=xxxxxxxx
ZOOM_CLIENT_ID=xxxxxxxx
ZOOM_CLIENT_SECRET=xxxxxxxx
```

These credentials are required for Laravel integration.

---

# 6. Add Required Scopes

From left sidebar:

```text
Scopes
```

Click:

```text
Add Scopes
```

Add the following scopes:

```text
meeting:write:admin
meeting:read:admin
user:read:admin
```

Purpose:

| Scope               | Purpose                 |
| ------------------- | ----------------------- |
| meeting:write:admin | Create meetings         |
| meeting:read:admin  | Read meetings           |
| user:read:admin     | Access user information |

#### Please verify that the scope below has been checked. 

[Open](https://marketplace.zoom.us/develop/apps/wnufEQL6Te-_Re8vZb7G_A/scope)

- ✅ meeting:write:registrant:master
- ✅ webinar:write:registrant:master
---

# 7. Activate App

From left sidebar:

```text
Activation
```

Click:

```text
Activate Your App
```

Without activation:

* Access token will fail
* Meeting creation will fail

---

# 8. Laravel Environment Configuration

Add into `.env`

```env
ZOOM_ACCOUNT_ID=
ZOOM_CLIENT_ID=
ZOOM_CLIENT_SECRET=

ZOOM_DEFAULT_TIMEZONE=Asia/Dhaka
```

---

# 9. Configure services.php

File:

```text
config/services.php
```

Add:

```php
'zoom' => [
    'account_id' => env('ZOOM_ACCOUNT_ID'),
    'client_id' => env('ZOOM_CLIENT_ID'),
    'client_secret' => env('ZOOM_CLIENT_SECRET'),
    'timezone' => env('ZOOM_DEFAULT_TIMEZONE', 'UTC'),
],
```

---

# 10. Clear Config Cache

Run:

```bash
php artisan config:clear
php artisan cache:clear
```

---

# 11. Create Migration

Command:

```bash
php artisan make:migration create_zoom_meetings_table
```

Migration:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('zoom_meetings', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('zoom_meeting_id')->unique();
            $table->string('topic');
            $table->text('agenda')->nullable();
            $table->string('join_url');
            $table->text('start_url');
            $table->string('password')->nullable();
            $table->timestamp('start_time');
            $table->integer('duration');
            $table->string('timezone');
            $table->string('status')->nullable();
            $table->json('meta')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('zoom_meetings');
    }
};
```

Run migration:

```bash
php artisan migrate
```

---

# 12. Create Model

Command:

```bash
php artisan make:model ZoomMeeting
```

Model:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class ZoomMeeting extends Model
{
    protected $fillable = [
        'user_id',
        'zoom_meeting_id',
        'topic',
        'agenda',
        'join_url',
        'start_url',
        'password',
        'start_time',
        'duration',
        'timezone',
        'status',
        'meta',
    ];

    protected $casts = [
        'start_time' => 'datetime',
        'meta'       => 'array',
    ];
}
```

---

# 13. Create Request Validation

Command:

```bash
php artisan make:request CreateZoomMeetingRequest
```

Request:

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class CreateZoomMeetingRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
         return [
            'topic'      => ['required', 'string', 'max:255'],
            'agenda'     => ['nullable', 'string', 'max:5000'],
            'start_time' => ['nullable', 'date', 'after:now'],
            'duration'   => ['required', 'integer', 'min:1', 'max:1440'],
            'timezone'   => ['nullable', 'timezone'],
        ];
    }
}
```

---

# 14.1. Create Zoom Service

Create file:

```text
app/Services/ZoomService.php
```

Service:

```php
<?php

<?php
namespace App\Services;

use App\Models\User;
use App\Models\ZoomMeeting;
use App\Traits\ApiResponse;
use Carbon\Carbon;
use Illuminate\Http\Exceptions\HttpResponseException;
use Illuminate\Support\Facades\Http;

class ZoomService
{
    use ApiResponse;
    protected function getAccessToken(): string
    {
        $response = Http::asForm()
            ->withBasicAuth(config('services.zoom.client_id'), config('services.zoom.client_secret'))
            ->post('https://zoom.us/oauth/token', [
                'grant_type' => 'account_credentials',
                'account_id' => config('services.zoom.account_id'),
            ]);

        if (! $response->successful()) {
            throw new HttpResponseException($this->error(null, $response->json('message'), $response->status()));
        }

        return $response->json('access_token');
    }

    public function createMeeting(User $user, array $data)
    {
        $token    = $this->getAccessToken();
        $timezone = $data['timezone'] ?? config('services.zoom.timezone');

        $response = Http::withToken($token)
            ->post('https://api.zoom.us/v2/users/me/meetings', [

                'topic'      => $data['topic'],
                'type'       => 2,

                'start_time' => ! empty($data['start_time'])
                    ? Carbon::parse($data['start_time'])->toISOString()
                    : now()
                    ->setTimezone($timezone)
                    ->addHour()
                    ->toISOString(),

                'duration'   => $data['duration'],
                'timezone'   => $timezone,
                'agenda'     => $data['agenda'] ?? null,
                'settings'   => [
                    'host_video'        => true,
                    'participant_video' => true,
                    'join_before_host'  => false,
                    'mute_upon_entry'   => true,
                    'waiting_room'      => true,
                    'auto_recording'    => 'none',
                ],
            ]);

        if (! $response->successful()) {
            throw new HttpResponseException($this->error(null, $response->json('message'), $response->status()));
        }

        $meeting = $response->json();

        return ZoomMeeting::create([
            'user_id'         => $user->id,
            'zoom_meeting_id' => $meeting['id'],
            'topic'           => $meeting['topic'],
            'agenda'          => $data['agenda'] ?? null,
            'join_url'        => $meeting['join_url'],
            'start_url'       => $meeting['start_url'],
            'password'        => $meeting['password'] ?? null,
            'start_time'      => $meeting['start_time'],
            'duration'        => $meeting['duration'],
            'timezone'        => $meeting['timezone'],
            'status'          => $meeting['status'] ?? null,
            'meta'            => $meeting,
        ]);
    }
}

```
---

# 14.2 Additional Setting

| Setting                             | Description                                              |
| ----------------------------------- | -------------------------------------------------------- |
| `host_video`                        | Host camera auto ON when meeting starts                  |
| `participant_video`                 | Participant camera auto ON when joining                  |
| `join_before_host`                  | Allow participants to join before host                   |
| `waiting_room`                      | Keep users in waiting room until host admits             |
| `mute_upon_entry`                   | Automatically mute participants on join                  |
| `auto_recording`                    | Automatically record the meeting                         |
| `approval_type`                     | Controls registration approval system                    |
| `audio`                             | Defines meeting audio type (`voip`, `telephony`, `both`) |
| `enforce_login`                     | Require Zoom login before joining                        |
| `allow_multiple_devices`            | Allow same user from multiple devices                    |
| `meeting_authentication`            | Require authenticated Zoom users only                    |
| `watermark`                         | Add watermark on participant screens                     |
| `breakout_room`                     | Enable breakout rooms                                    |
| `focus_mode`                        | Limit participant visibility during meeting              |
| `show_share_button`                 | Allow participants to share meeting invite               |
| `registrants_email_notification`    | Send registration emails automatically                   |
| `waiting_room_options`              | Configure waiting room behavior                          |
| `continuous_meeting_chat`           | Enable persistent meeting chat                           |
| `device_testing`                    | Allow device/audio testing before join                   |
| `private_meeting`                   | Mark meeting as private                                  |
| `email_notification`                | Enable Zoom email notifications                          |
| `auto_start_meeting_summary`        | Automatically generate AI meeting summary                |
| `auto_start_ai_companion_questions` | Enable AI companion during meeting                       |

### Uses

```php
'settings' => [

            'host_video' => true,             // Host camera ON/OFF
            'participant_video' => true,      // Participant camera ON/OFF
            'join_before_host' => false,      // Allow participant before host joins
            'mute_upon_entry' => true,        // Auto mute participants on join
            'waiting_room' => true,           // Waiting room enable/disable
            'auto_recording' => 'none',       // Auto recording    options: none | local | cloud
            'audio' => 'voip',                // Meeting audio type options: both | telephony | voip
            'enforce_login' => false,         // Require Zoom login before joining
            'allow_multiple_devices' => false,// Allow multiple device join
            'watermark' => false,             // Watermark enable           
            'approval_type' => 2,             // Registration approval type   0 = auto approve   1 = manual approve      2 = no registration required
            'focus_mode' => false,            // Enable focus mode

            'continuous_meeting_chat' => ['enable' => true,],            // Enable meeting chat
            'breakout_room' => [ 'enable' => false,],                    // Enable breakout room
        ],
```

# 15. Create Controller

Command:

```bash
php artisan make:controller Api/ZoomMeetingController
```

Controller:

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\CreateZoomMeetingRequest;
use App\Services\ZoomService;
use Illuminate\Support\Facades\Auth;

class ZoomMeetingController extends Controller
{
     public function store(CreateZoomMeetingRequest $request, ZoomService $zoomService)
    {
        $meeting = $zoomService->createMeeting(Auth::user(), $request->validated());

        return response()->json([
            'success' => true,
            'message' => 'Zoom meeting created successfully.',
            'data' => $meeting,
        ]);
    }
}
```

---

# 16. API Route

File:

```text
routes/api.php
```

Route:

```php
use App\Http\Controllers\Api\ZoomMeetingController;

Route::middleware('auth:sanctum')->group(function () {

    Route::post('/zoom/meetings', [ZoomMeetingController::class, 'store']);
}
```

---

# 17. Postman API Example

## Endpoint

```text
POST /api/zoom/meetings
```

---

## Headers

```text
Authorization: Bearer YOUR_TOKEN
Accept: application/json
Content-Type: application/json
```

---

## Request Body

```json
{
    "topic": "Project Discussion",
    "agenda": "Weekly Development Meeting",
    "start_time": "2026-05-15 10:00:00", // optional
    "duration": 60,
    "timezone": "Asia/Dhaka"
}
```

---

# 18. Successful Response Example

```json
{
    "success": true,
    "message": "Zoom meeting created successfully.",
    "data": {
        "id": 1,
        "user_id": 1,
        "zoom_meeting_id": "87544522111",
        "topic": "Project Discussion",
        "agenda": "Weekly Development Meeting",
        "join_url": "https://zoom.us/j/87544522111",
        "start_url": "https://zoom.us/s/87544522111",
        "password": "123456",
        "start_time": "2026-05-15T04:00:00Z",
        "duration": 60,
        "timezone": "Asia/Dhaka",
        "status": "waiting"
    }
}
```

---

# 19. Important Security Guidelines

## Never expose `start_url`

`start_url` contains host access.

Only admin/host should access it.

---

## Always store credentials in `.env`

Never hardcode secrets inside source code.

---

## Always validate request data

Use Form Request validation.

---

## Use Service Layer

Recommended architecture:

```text
Controller
   ↓
Service
   ↓
Zoom API
   ↓
Database
```

---

## Log API errors

Always log Zoom API failures for debugging.

Example:

```php
Log::error($response->json());
```

---

# 20. Recommended Future Improvements

You can later implement:

* Update Meeting API
* Delete Meeting API
* Zoom Webhook Integration
* Participant Tracking
* Meeting Recording Sync
* Queue-based meeting creation
* Retry mechanism
* Cached Zoom tokens
* Host authorization
* Multi-user Zoom support

---

# 21. Useful Official Documentation

* [Zoom API Documentation](https://developers.zoom.us/docs/api/)
* [Zoom Server-to-Server OAuth Guide](https://developers.zoom.us/docs/internal-apps/s2s-oauth/)
* [Zoom Marketplace](https://marketplace.zoom.us/)


## Author

Developed and maintained by [MD. Rahatul Rabbi](https://github.com/learnwithfair).

---

## Follow

[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/github.svg' alt='github' height='30'>](https://github.com/learnwithfair)
[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/facebook.svg' alt='facebook' height='30'>](https://www.facebook.com/learnwithfair/)
[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/instagram.svg' alt='instagram' height='30'>](https://www.instagram.com/learnwithfair/)
[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/youtube.svg' alt='YouTube' height='30'>](https://www.youtube.com/@learnwithfair)

---

<!-- MARKDOWN LINKS -->
[youtube-shield]: https://img.shields.io/badge/-Youtube-black.svg?style=flat-square&logo=youtube&color=555&logoColor=white
[youtube-url]: https://youtube.com/@learnwithfair
[facebook-shield]: https://img.shields.io/badge/-Facebook-black.svg?style=flat-square&logo=facebook&color=555&logoColor=white
[facebook-url]: https://facebook.com/learnwithfair
[instagram-shield]: https://img.shields.io/badge/-Instagram-black.svg?style=flat-square&logo=instagram&color=555&logoColor=white
[instagram-url]: https://instagram.com/learnwithfair
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=flat-square&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/company/learnwithfair

#learnwithfair #rahtulrabbi #rahatul-rabbi #learn-with-fair
