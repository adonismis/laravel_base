# 基本流程

## 安裝Laravel

laravel new contact_manager 或 composer create-project --prefer-dist laravel/laravel contact_manager
cd contact_manager

## 定義Migration
php artisan make:migration create_group_and_contact_table --create=groups

```
    public function up()
    {
        Schema::create('groups', function (Blueprint $table) {
            $table->increments('id');
            $table->string('name');
            $table->timestamps();
        });

        Schema::create('contact', function (Blueprint $table) {
            $table->increments('id');
            $table->integer('group_id')->unsigned()->default(0);
            $table->foreign('group_id')->references('id')->on('group')->onDelete('cascade');
            $table->string('name');
            $table->string('company');
            $table->string('email');
            $table->string('phone');
            $table->string('address');
            $table->timestamps();
        });
    }

   
    public function down()
    {
        Schema::drop('groups');
        Schema::drop('contacts');
    }
```
php artisan migrate


## Database Seeding

php artisan make:seeder GroupTableSeeder
```
public function run()
    {
      DB::statement('SET FOREIGN_KEY_CHECKS = 0');
        DB::table('groups')->truncate();
        $groups = [
            ['id' => 1,'name'=>'Family','created_at'=>new DateTime,'updated_at'=>new DateTime],
            ['id' => 2,'name'=>'Family1','created_at'=>new DateTime,'updated_at'=>new DateTime],
            ['id' => 3,'name'=>'Family2','created_at'=>new DateTime,'updated_at'=>new DateTime],
            ['id' => 4,'name'=>'Family3','created_at'=>new DateTime,'updated_at'=>new DateTime],
            ['id' => 5,'name'=>'Family4','created_at'=>new DateTime,'updated_at'=>new DateTime],
            ['id' => 6,'name'=>'Family5','created_at'=>new DateTime,'updated_at'=>new DateTime]
        ];
        DB::table('groups')->insert($groups);
     DB::statement('SET FOREIGN_KEY_CHECKS = 1');
    }
```

php artisan make:seeder ContactTableSeeder
```
use Illuminate\Database\Seeder;
use Faker\Factory as Faker;

class ContactTableSeeder extends Seeder
{
    public function run()
    {
      DB::statement('SET FOREIGN_KEY_CHECKS = 0');
        DB::table('contact')->truncate();
        $faker = Faker::create();
        $contact= [];

        foreach (range(1,20) as $index) {
            $contacts[] = [
                'name'  => $faker->name,
                'email' => $faker->email,
                'phone' => $faker->phoneNumber,
                'address' => "{$faker->streetName}",
                'company' => $faker->company,
                'created_at' => new DateTime,
                'updated_at' => new DateTime ,
                'group_id' => rand(1,3)
            ];
        }
        DB::table('contacts')->insert($contact);
       DB::statement('SET FOREIGN_KEY_CHECKS = 1');
    }
}
```


/app/database/seeds/DatabaseSeeder.php
```
    public function run()
    {
       $this->call(ContactTableSeeder::class);
       $this->call(GroupTableSeeder::class);
    }
```
php artisan db:seed

## Route
```
     Route:resource(‘contact’,’ContactController’);
```

## Model

php artisan make:model Group
```
<?php

namespace App;

use Illuminate\Database\Eloquent\Model;

class Contact extends Model
{
    protected $fillable = [
        'name', 'email', 'address', 'company', 'phone', 'group_id', 'photo', 'user_id'
    ];

    public function group()
	{
		return $this->belongsTo('App\Group');
	}

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

## Controller

php artisan make:controller ContactController

```
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

use App\Http\Requests;
use App\Contact;

class ContactsController extends Controller
{
    private $limit = 5;
    private $rules = [
        'name' => ['required', 'min:5'],
        'company' => ['required'],
        'email' => ['required', 'email'],
        'photo' => ['mimes:jpg,jpeg,png,gif,bmp']
    ];
    private $upload_dir = 'public/uploads';

    public function __construct()
    {
        $this->middleware('auth');
        $this->upload_dir = base_path() . '/' . $this->upload_dir;
    }

    public function autocomplete(Request $request)
    {
        if ($request->ajax())
        {
            return  Contact::select(['id', 'name as value'])->where(function($query) use ($request) {
                                if ( ($term = $request->get("term")) )
                                {
                                    $keywords = '%' . $term . '%';
                                    $query->orWhere("name", 'LIKE', $keywords);
                                    $query->orWhere("company", 'LIKE', $keywords);
                                    $query->orWhere("email", 'LIKE', $keywords);
                                }
                            })
                            ->orderBy('name', 'asc')
                            ->take(5)
                            ->get();
        }
    }

    public function index(Request $request)
    {
        $contacts = Contact::where(function($query) use ($request) {
                            // filter by current user
                            $query->where("user_id", $request->user()->id);

                            if ($group_id = ($request->get('group_id'))) {
                                $query->where('group_id', $group_id);
                            }

                            if ( ($term = $request->get("term")) )
                            {
                                $keywords = '%' . $term . '%';
                                $query->orWhere("name", 'LIKE', $keywords);
                                $query->orWhere("company", 'LIKE', $keywords);
                                $query->orWhere("email", 'LIKE', $keywords);
                            }
                        })
                        ->orderBy('id', 'desc')
                        ->paginate($this->limit);

    	return view('contacts.index', compact('contacts'));
    }

    public function create()
    {
        return view("contacts.create");
    }

    public function edit($id)
    {
        $contact = Contact::findOrFail($id);
        $this->authorize('modify', $contact);
        return view("contacts.edit", compact('contact'));
    }

    public function store(Request $request)
    {
        $this->validate($request, $this->rules);

        $data = $this->getRequest($request);

        $request->user()->contacts()->create($data);

        return redirect('contacts')->with('message', 'Contact Saved!');
    }

    private function getRequest(Request $request)
    {
        $data = $request->all();

        if ($request->hasFile('photo'))
        {
            $photo       = $request->file('photo');
            $fileName    = $photo->getClientOriginalName();
            $destination = $this->upload_dir;
            $photo->move($destination, $fileName);

            $data['photo'] = $fileName;
        }

        return $data;
    }

    public function update($id, Request $request)
    {
        $contact = Contact::findOrFail($id);
        $this->authorize('modify', $contact);

        $this->validate($request, $this->rules);

        $oldPhoto = $contact->photo;

        $data = $this->getRequest($request);
        $contact->update($data);

        if ($oldPhoto !== $contact->photo) {
            $this->removePhoto($oldPhoto);
        }

        return redirect('contacts')->with('message', 'Contact Updated!');
    }

    public function destroy($id)
    {
        $contact = Contact::findOrFail($id);
        $this->authorize('modify', $contact);

        $contact->delete();

        $this->removePhoto($contact->photo);

        return redirect('contacts')->with('message', 'Contact Deleted!');
    }

    private function removePhoto($photo)
    {
        if ( ! empty($photo))
        {
            $file_path = $this->upload_dir . '/' . $photo;

            if ( file_exists($file_path) ) unlink($file_path);
        }
    }
}

```

## Value

### Layout 
- app.blade.php
```
@include('layouts.partials.header')

    @include('layouts.partials.navbar')

    @yield('content')

@include('layouts.partials.footer')
```

### contacts
- create.blade
```
@extends('layouts.main')

@section('content')

    <div class="panel panel-default">
            <div class="panel-heading">
              <strong>Add Contact</strong>
            </div>
            {!! Form::open(['route' => 'contacts.store', 'files' => true]) !!}
            
            @include("contacts.form")

            {!! Form::close() !!}
          </div>

@endsection
```

- edit.blade
```
@extends('layouts.main')

@section('content')

    <div class="panel panel-default">
            <div class="panel-heading">
              <strong>Edit Contact</strong>
            </div>
            {!! Form::model($contact, ['files' => true, 'route' => ['contacts.update', $contact->id], 'method' => 'PATCH']) !!}

            @include("contacts.form")

            {!! Form::close() !!}
          </div>

@endsection
```

- form.blade
``` 
<div class="panel-body">
  <div class="form-horizontal">
    <div class="row">
      <div class="col-md-8">
          @if (count($errors))
              <div class="alert alert-danger">
                  <ul>
                      @foreach ($errors->all() as $error)
                          <li>{{ $error }}</li>
                      @endforeach
                  </ul>
              </div>
          @endif
        <div class="form-group">
          <label for="name" class="control-label col-md-3">Name</label>
          <div class="col-md-8">
            {!! Form::text('name', null, ['class' => 'form-control']) !!}
          </div>
        </div>

        <div class="form-group">
          <label for="company" class="control-label col-md-3">Company</label>
          <div class="col-md-8">
            {!! Form::text('company', null, ['class' => 'form-control']) !!}
          </div>
        </div>

        <div class="form-group">
          <label for="email" class="control-label col-md-3">Email</label>
          <div class="col-md-8">
            {!! Form::text('email', null, ['class' => 'form-control']) !!}
          </div>
        </div>

        <div class="form-group">
          <label for="phone" class="control-label col-md-3">Phone</label>
          <div class="col-md-8">
            {!! Form::text('phone', null, ['class' => 'form-control']) !!}
          </div>
        </div>

        <div class="form-group">
          <label for="name" class="control-label col-md-3">Address</label>
          <div class="col-md-8">
            {!! Form::textarea('address', null, ['class' => 'form-control', 'rows' => 2]) !!}
          </div>
        </div>
        <div class="form-group">
          <label for="group" class="control-label col-md-3">Group</label>
          <div class="col-md-5">
            {!! Form::select('group_id', App\Group::pluck('name', 'id'), null, ['class' => 'form-control']) !!}
          </div>
          <div class="col-md-3">
            <a href="#" id="add-group-btn" class="btn btn-default btn-block">Add Group</a>
          </div>
        </div>
        <div class="form-group" id="add-new-group" style="display: none;">
          <div class="col-md-offset-3 col-md-8">
            <div class="input-group">
              <input type="text" name="new_group" id="new_group" class="form-control">
              <span class="input-group-btn">
                <a href="#" id="add-new-btn" class="btn btn-default">
                  <i class="glyphicon glyphicon-ok"></i>
                </a>
              </span>
            </div>
          </div>
        </div>
      </div>
      <div class="col-md-4">
        <div class="fileinput fileinput-new" data-provides="fileinput">
          <div class="fileinput-new thumbnail" style="width: 150px; height: 150px;">
              <?php $photo = ! empty($contact->photo) ? $contact->photo : 'default.png' ?>
              {!! Html::image('uploads/' . $photo, "Choose photo", ['width' => 150, 'height' => 150]) !!}
          </div>
          <div class="fileinput-preview fileinput-exists thumbnail" style="max-width: 200px; max-height: 150px;"></div>
          <div class="text-center">
            <span class="btn btn-default btn-file"><span class="fileinput-new">Choose Photo</span><span class="fileinput-exists">Change</span>{!! Form::file('photo') !!}</span>
            <a href="#" class="btn btn-default fileinput-exists" data-dismiss="fileinput">Remove</a>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
<div class="panel-footer">
  <div class="row">
    <div class="col-md-8">
      <div class="row">
        <div class="col-md-offset-3 col-md-6">
          <button type="submit" class="btn btn-primary">{{ ! empty($contact->id) ? "Update" : "Save"}}</button>
          <a href="#" class="btn btn-default">Cancel</a>
        </div>
      </div>
    </div>
  </div>
</div>

@section('form-script')
    <script>
    $("#add-new-group").hide();
    $('#add-group-btn').click(function () {
      $("#add-new-group").slideToggle(function() {
        $('#new_group').focus();
      });
      return false;
    });

    $("#add-new-btn").click(function() {
        var newGroup = $("#new_group");
        var inputGroup = newGroup.closest('.input-group');

        $.ajax({
            url: "{{ route("groups.store") }}",
            method: 'post',
            data: {
                name: $("#new_group").val(),
                _token: $("input[name=_token]").val()
            },
            success: function(group) {
                if (group.id != null) {
                    inputGroup.removeClass('has-error');
                    inputGroup.next('.text-danger').remove();

                    var newOption = $('<option></option>')
                        .attr('value', group.id)
                        .attr('selected', true)
                        .text(group.name);
                        
                    $("select[name=group_id]")
                        .append( newOption );

                    newGroup.val("");
                }
            },
            error: function(xhr) {
                var errors = xhr.responseJSON;
                var error = errors.name[0];
                if (error) {
                    inputGroup.next('.text-danger').remove();

                    inputGroup
                        .addClass('has-error')
                        .after('<p class="text-danger">' + error + '</p>');
                }
            }
        });
    });
    </script>
@endsection
```

- index.blade
```
@extends('layouts.main')

@section('content')
    <div class="panel panel-default">
        <div class="panel-heading clearfix">
            <div class="pull-left">
                <h4>All Contacts</h4>
            </div>
            <div class="pull-right">
                <a href="{{ route("contacts.create") }}" class="btn btn-success">
                  <i class="glyphicon glyphicon-plus"></i>
                  Add Contact
                </a>
            </div>
        </div>
        
      <table class="table">

          @foreach($contacts as $contact)

            <tr>
              <td class="middle">
                <div class="media">
                  <div class="media-left">
                    <a href="#">

                        <?php $photo = ! is_null($contact->photo) ? $contact->photo : 'default.png' ?>
                        {!! Html::image('uploads/' . $photo, $contact->name, ['class' => 'media-object', 'width' => 100, 'height' => 100]) !!}

                    </a>
                  </div>
                  <div class="media-body">
                      <h4 class="media-heading">{{ $contact->name }}</h4>
                      <address>
                        <strong>{{ $contact->company }}</strong><br>
                        {{ $contact->email }}
                      </address>
                  </div>
                </div>
              </td>
              <td width="100" class="middle">
                <div>
                    {!! Form::open(['method' => 'DELETE', 'route' => ['contacts.destroy', $contact->id]]) !!}
                      <a href="{{ route("contacts.edit", ['id' => $contact->id]) }}" class="btn btn-circle btn-default btn-xs" title="Edit">
                        <i class="glyphicon glyphicon-edit"></i>
                      </a>
                      <button onclick="return confirm('Are you sure?');" type="submit" class="btn btn-circle btn-danger btn-xs" title="Edit">
                        <i class="glyphicon glyphicon-remove"></i>
                      </button>
                    {!! Form::close() !!}
                </div>
              </td>
            </tr>

        @endforeach

      </table>
    </div>

    <div class="text-center">
      <nav>
        {!! $contacts->appends( Request::query() )->render() !!}
      </nav>
    </div>
@endsection
```








