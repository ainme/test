### First Scenario - Belongs to many through many

Example

-  user belongs to many roles
-  roles belongs to many users
-  roles belongs to many permissions
-  permissions belongs to many roles
-  users → user-role → roles → role-permission→ permissions
-  5 tables (2 pivots)
     
### So, Lets say we want to fetch a users permissions, which are connected through roles

```php
 //attach roles to user
$user->roles()->attach($admin->id);
//attach permissions to user
$admin->permissions()->attach($permission->id);
//get users' permissions using HasManyThroughMany relationship
$userPermissions = $user->permissions();
```

User.php


```php
use App\Extensions\Eloquent\Model as Model;

class User extends Model
{
    public function roles()
    {
        return $this->belongsToMany(Role::class, 'user_role', 'user_id', 'role_id');
    }

    public function permissions()
    {        
        return $this->hasManyThroughMany(Permission::class,
										 Role::class,
										 'user_role',
										 'role_permission',
										 'permission_id',
										 'role_id');
    }
}
```

Role.php

```php
class Role extends Model
{	
    public function users()
    {
        return $this->belongsToMany(User::class, 'user_role', 'role_id', 'user_id');
    }

    public function permissions()
    {
        return $this->belongsToMany(Permission::class, 'role_permission', 'role_id', 'permission_id');
    }
}
```

Permission.php

```php
class Permission extends Model
{		
    public function users()
    {
        return $this->belongsToMany(User::class ,'user_permission', 'permission_id', 'user_id');
    }

    public function roles()
    {
        return $this->belongsToMany(Role::class, 'role_permission', 'permission_id', 'role_id');
    }
}
```
### Second Scenario - Has many through many

Example

-  student has many enrollments
-  each enrollment has many subjects
-  each subject belongs to an enrollment
-  each enrollment belongs to a student
-  student→ enrollment → subject
-  3 tables

#### So, Lets say we want to fetch a students subjects, which are connected through enrollments

```php
//save students enrollment
$student->enrollments()->save($enrollment);
//save enrollments subject
$enrollment->subjects()->save($subject);
//get student's subjects using HasManyThroughMany relationship
$student->subjects();
```

		

Student.php

```php
use App\Extensions\Eloquent\Model as Model;

class Student extends Model
{
    public function enrollments()
    {
    	return $this->hasMany(Enrollment::class);
    }

    public function subjects()
    {
    	return $this->hasManyThroughMany(Subject::class, Enrollment::class);
    }
}
```

Enrollment.php

```php
class Enrollment extends Model
{
    public function subjects()
    {
    	return $this->hasMany(Subject::class);
    }

    public function student()
    {
    	return $this->belongsTo(Student::class);
    }
}
```

Subject.php

```php
class Subject extends Model
{
    public function enrollment()
    {
    	return $this->belongsTo(Enrollment::class);
    }
}
```



##### New HasManyThroughMany relationship file.

HasManyThroughMany.php
```php
namespace App\Extensions\Eloquent;

use Illuminate\Database\Eloquent\Model as EloquentModel;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Query\Expression;
use Illuminate\Database\Eloquent\Collection;
use Illuminate\Database\Eloquent\Relations\Relation;

class HasManyThroughMany extends Relation
{
    public function __construct(
		Builder $query,
		EloquentModel $farParent,
		EloquentModel $parent,
		$firstKey, $secondKey,
		$localKey,
		$throughKey)
    {
        $this->localKey = $localKey;
        $this->firstKey = $firstKey;
        $this->secondKey = $secondKey;
        $this->farParent = $farParent;
        $this->throughKey = $throughKey;
        $this->farParentRelatedKey = $this->getModelRelatedKey($this->farParent);
        parent::__construct($query, $parent);
    }

    public function addConstraints()
    {
        $parentTable = $this->parent->getTable();

        $this->setJoin();
        if (static::$constraints && !$this->throughKey) {
            $localValue = $this->farParent[$this->localKey];
            $this->query->where($parentTable.'.'.$this->firstKey, '=', $localValue);
        } else {
            $localValue = $this->farParent['id'];
            $this->query->where($this->firstKey . '.' . $this->farParentRelatedKey, '=', $localValue);
        }

    }

    protected function setJoin(Builder $query = null)
    {

        $query = $query ?: $this->query;

        $foreignKey = $this->related->getTable().'.'.$this->secondKey;
        $farParentTable = $this->farParent->getTable();
        $farParentTableKey = $farParentTable.'.'.$this->localKey;
        $firstKey = $this->parent->getTable().'.'.$this->firstKey;

        $id = $this->related->getTable() . '.id as id';
        $all = $this->related->getTable() . '.*';
        $columns = [$id, $all];
        if(!$this->throughKey){
            $query->addSelect($columns)
			->join($this->parent->getTable(),
					$this->getQualifiedParentKeyName(),
					'=',
					$foreignKey)
            ->join($farParentTable,
					$farParentTableKey,
					'=',
					$firstKey);
        } else {
            $query->addSelect($columns)
            ->join($this->secondKey,
					$this->secondKey . '.' . $this->localKey,
					'=',
					$this->related->getTable() . '.id')
            ->join($this->firstKey,
					$this->firstKey . '.' . $this->throughKey,
					'=',
					$this->secondKey . '.' . $this->throughKey)
            ->join($farParentTable,
					$farParentTable . '.id',
					'=',
					$this->firstKey . '.' . $this->farParentRelatedKey);
        }
        if ($this->parentSoftDeletes()) {
            $query->whereNull($this->parent->getQualifiedDeletedAtColumn());
        }

    }

    public function getModelRelatedKey($model)
    {
        return strtolower(class_basename($model)) . '_id';
    }

    public function getForeignKey()
    {
        return $this->related->getTable().'.'.$this->secondKey;
    }

    public function parentSoftDeletes()
    {
        return in_array('Illuminate\Database\Eloquent\SoftDeletes',
						class_uses_recursive(get_class($this->parent)));
    }

    public function addEagerConstraints(array $models){}
    public function initRelation(array $models, $relation)
    {
        foreach ($models as $model) {
            $model->setRelation($relation, $this->related->newCollection());
        }

        return $models;
    }
    public function match(array $models, Collection $results, $relation){}
    public function getResults()
    {
        return $this->get();
    }    
}
```

##### New Model file

Model.php

```php
namespace Hello\Extensions\Eloquent;

use Illuminate\Database\Eloquent\Model AS EloquentModel;
use App\Extensions\Eloquent\HasManyThroughMany;

class Model extends EloquentModel
{
    public function hasManyThroughMany($related,
									   $through,
									   $firstKey = null,
									   $secondKey = null,
									   $localKey = null,
									   $throughKey = null)
    {
        $through = new $through;

        $firstKey = $firstKey ?: $this->getForeignKey();

        $secondKey = $secondKey ?: $through->getForeignKey();

        $localKey = $localKey ?: $this->getKeyName();

        return new HasManyThroughMany((new $related)->newQuery(),
										$this,
										$through,
										$firstKey,
										$secondKey,
										$localKey,
										$throughKey);

    }

}
```
