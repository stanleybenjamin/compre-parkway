# Compre Parkway

A Laravel package for facial biometrics — enrollment, recognition, verification, and detection — backed by [CompreFace](https://github.com/exadel-inc/CompreFace) or AWS Rekognition.

## Requirements

- PHP 8.0+
- Laravel 8.75+

## Installation

```bash
composer require stanliwise/compre-parkway
```

Publish the config file:

```bash
php artisan vendor:publish --tag=compreface-config
```

Run the migrations:

```bash
php artisan migrate
```

## Configuration

Add the following variables to your `.env` file.

**CompreFace driver:**

```env
COMPREFACE_DRIVER=compreFace
COMPREFACE_BASE_URL=http://localhost:8000
COMPREFACE_RECOGNITION_API_KEY=your-recognition-api-key
COMPREFACE_VERIFY_API_KEY=your-verification-api-key
COMPREFACE_DETECT_API_KEY=your-detection-api-key
COMPREFACE_TRUST_THRESHOLD=0.8
COMPREFACE_SAVE_DIRECTORY=faces
COMPREFACE_STORAGE_DRIVE=local
```

**AWS Rekognition driver:**

```env
COMPREFACE_DRIVER=aws
AWS_ACCESS_KEY_ID=your-access-key-id
AWS_SECRET_ACCESS_KEY=your-secret-access-key
AWS_DEFAULT_REGION=us-east-1
AWS_COLLECTION_ID=your-collection-id
COMPREFACE_TRUST_THRESHOLD=0.8
```

## Usage

### 1. Prepare Your Model

Add the `HasFacialBiometrics` trait and implement the `Subject` contract on any Eloquent model that needs facial biometrics (e.g. `User`).

```php
use Stanliwise\CompreParkway\Contract\Subject;
use Stanliwise\CompreParkway\Traits\HasFacialBiometrics;

class User extends Authenticatable implements Subject
{
    use HasFacialBiometrics;

    public function getUniqueID()
    {
        return (string) $this->id;
    }

    public function setfacialUUID(string $uuid)
    {
        $this->facial_uuid = $uuid;
        $this->save();
    }
}
```

> Your `users` table needs a `facial_uuid` column (or whichever column you store the UUID in).

### 2. Wrapping an Image

All image inputs must be wrapped in an `ImageFile` instance, which implements the `File` contract:

```php
use Stanliwise\CompreParkway\Adaptors\File\ImageFile;

$file = new ImageFile('/path/to/photo.jpg');
$file->setTag('front-facing'); // optional label stored with the record
```

---

### Enrolling a Subject

Enrollment registers a subject's primary face image with the facial recognition service.

```php
use Stanliwise\CompreParkway\Facade\FaceTech;
use Stanliwise\CompreParkway\Adaptors\File\ImageFile;

$user = User::find(1);
$image = new ImageFile(storage_path('app/faces/john.jpg'));
$image->setTag('profile');

FaceTech::enroll($user, $image, disk_drive: 'local');
```

Throws `SubjectAlreadyEnrolled` if the subject has already been enrolled.

---

### Adding Extra Face Images

After enrollment, you can attach additional face samples to improve recognition accuracy:

```php
$user = User::find(1);
$image = new ImageFile(storage_path('app/faces/john-side.jpg'));
$image->setTag('side-profile');

FaceTech::addSecondaryExample($user, $image, disk: 'local');
// or via the trait shorthand:
$user->addFaceImage($image);
```

---

### Verifying a Face Against a Subject

Check whether a given image matches the enrolled face of a specific subject:

```php
$user = User::find(1);
$image = new ImageFile(storage_path('app/uploads/attempt.jpg'));
$image->setTag('login-attempt');

$result = FaceTech::verifyFaceImageAgainstASubject($user, $image);

// $result contains similarity score and match details from the driver
```

---

### Comparing Two Images

Compare any two face images directly, without requiring an enrolled subject:

```php
$source = new ImageFile(storage_path('app/faces/source.jpg'));
$target = new ImageFile(storage_path('app/faces/target.jpg'));

$source->setTag('source');
$target->setTag('target');

$result = FaceTech::compareTwoFileImages($source, $target);
```

---

### Detecting a Face in an Image

Confirm that a face is present in an image and retrieve detection metadata:

```php
$image = new ImageFile(storage_path('app/uploads/photo.jpg'));
$image->setTag('upload');

$result = FaceTech::detectFileImage($image);
```

Throws `NoFaceWasDetected` if no face is found in the image.

---

### Disenrolling a Subject

Remove a subject and all their associated face data:

```php
FaceTech::disenroll($user);
```

### Removing Individual Examples

```php
// Remove one face example (primary examples cannot be removed individually)
FaceTech::removeExample($example);

// Remove all secondary examples, keeping the subject enrolled
FaceTech::removeAllExamples($user);
```

---

## Switching Drivers at Runtime

The driver is resolved from config by default, but you can swap it per-request:

```php
use Stanliwise\CompreParkway\Adaptors\AwsFacialAdaptor;
use Stanliwise\CompreParkway\Facade\FaceTech;

FaceTech::setDriver(new AwsFacialAdaptor());

$result = FaceTech::verifyFaceImageAgainstASubject($user, $image);
```

---

## Opting Out of Migrations

If you manage migrations yourself, call this in your `AppServiceProvider`:

```php
use Stanliwise\CompreParkway\CompreFaceServiceProvider;

CompreFaceServiceProvider::ignoreMigrations();
```

---

## Exception Reference

| Exception | When it is thrown |
|---|---|
| `SubjectAlreadyEnrolled` | `enroll()` called on an already-enrolled subject |
| `SubjectNameAlreadyExist` | Subject UUID collision during enrollment |
| `FaceHasNotBeenIndexed` | Recognition finds no matching subject for the image |
| `MultipleFaceDetected` | Recognition returns more than one matching subject |
| `NoFaceWasDetected` | Detection finds no face in the supplied image |
| `FaceDoesNotMatch` | Verification result falls below the trust threshold |
| `NoPrimaryImageWasFound` | Verification attempted without a primary enrolled image |

---

## Testing

```bash
composer test
```

## License

MIT
