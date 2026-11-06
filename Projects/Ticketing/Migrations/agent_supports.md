
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('support_agents', function (Blueprint $table) {
            $table->id();
            // $table->foreignId('invited_by_admin_id')->nullable()->constrained('admins')->cascadeOnDelete();
            // $table->timestamp('invitation_accepted_at')->nullable();
            $table->string('first_name', 100);
            $table->string('last_name', 100);
            $table->string('email')->unique();
            $table->string('password');
            $table->string('avatar_path')->nullable();
            $table->string('status', 20)->default('invited');
            $table->string('mobile')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->rememberToken();
            $table->timestamp('last_login_at')->nullable();
            $table->timestamps();
            $table->softDeletes();
            $table->index(['status', 'created_at']);
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('support_agents');
    }
};
```