Throwarm
======
Thowarm allows you to parse data sets like SQL results into re-useable, hierarchal objects.

#Why does this exist?
Throwarm seeks to gain some of the advantages of an ORM without the weight or complexity.  The idea is to allow developers to use normal SQL commands to interact with the database, but once results are returned, the developer can use throwarm to interact with the results set as a hierarchal set of objects.

#Overview
##Installing throwarm
to install throwarm use npm → npm install throwarm

##Using throwarm
Using throwarm is a three part process 1) creating throwarm objects, 2) creating a Datastructure, 3) applying data to the Datastructure

###throwarm objects
There are four different types of throwarm objects: Datagrams, Datasets, Calcs, and Extensions.   Datagrams are objects that hold a single result, analogous to a table row.  Datasets are array objects that hold sets of Datagrams,they are analogous to a table. Calcs execute a function during data population.  Extensions are throwarm objects that do not process any data.

####Datagrams
To create a Datagram, you create an function,  make a base call to the datagram constructor and then set the default field map.  The default field map is matches the fields in the database table the datagram represents.

For Example: 
```javascript
var Datagram = require('throwarm').Datagram;
function Parent() {
	Object.defineProperty(this,'base',{value:Datagram,writable:false,configurable:true,enumerable:false});

	this.base(this);
	this.fieldMap =  {
		id:'mainID',
		name: 'mainName',
		property1: 'mainProperty'
	};

}
Parent.prototype = new Datagram();
Parent.prototype.constructor = Parent;
```

####Datasets
Dataset creation uses slightly different syntax.  Since datasets do not have fields they do not have a field map.  They do however have a type name that should match the name of the Dataset.

For Example: 
```javascript
var Dataset = require('throwarm').Dataset;
function Children() {
	var children = new Dataset();
	children.typeName= 'Children';
	return children
}
```
####Calcs
Calculations are appended to the field map of a Datagram.  A calculation takes two arguments, an array of the fields to be used in the calculation and a function that returns the calculation value.

For Example:
```javascript
var Datagram = require('throwarm').Datagram;
var Calc = require('throwarm').Calc
function Parent() {
	Object.defineProperty(this,'base',{value:Datagram,writable:false,configurable:true,enumerable:false});

	this.base(this);
	this.fieldMap =  {
		id:'mainID',
		name: 'mainName',
		property1: 'mainProperty'
		fullname: new Calc(['firstName', 'lastName'], function(first, last) {
			return first + ' ' + last;
		});
	};

}
Parent.prototype = new Datagram();
Parent.prototype.constructor = Parent;
```
####Extensions
Extensions act like Datagrams but do not process data.  They are do not any required attributes.

For Example:
```javascript

var Extension = require('throwarm').Extension
function More() {
	this.hello = function() {return 'hi';}
}
More.prototype = new Extension();
Parent.prototype.constructor = More;
```

###Extending throwarm objects
One of the nifty things about throwarm is that it allows you to attach code to datasets and datagrams that act on the populated data.  

####Extending Datagrams
Once you have created the base datagram they can be extended by adding functions to the objects prototype.
For example the following code defines a Datagram and extends it.
```javascript
function Child() {
	Object.defineProperty(this,'base',{value:Datagram, writable:false,configurable:true,enumerable:false});

	this.base(this);
	this.fieldMap =  {
		id:'childID',
		name: 'childName',
		property1: 'childProperty',
		type: 'childType'
	};

}

Child.prototype = new Datagram();
Child.prototype.constructor = Child;


//here is a simple function 
Child.prototype.isFast = function() {
	var self = this;
	return self.property1 === 'Fast';
};
```
####Extending Datasets
Datasets must be extended within the definition function the following code extends the Children Dataset:
```javascript
function Children() {
	var children = new Dataset();
	childern.typeName= 'Children';

	//I am adding a function to the object
	function getByType(type) {
		var self = this;
		function byType(child) {
			return child.type === type;
		}
		return self.copyReturn(self.filter(byType), new Children());
	}
	Object.defineProperty(children,'getByType',{value:getByType,writable:false,configurable:false,enumerable:false});
	
	function getById(id) {
		var self = this;
		function byID(child) {
			return child.id == id;
		}
		var child = self.filter(byID);
		if(child.length === 0) {
			return null;
		}
		return child[0];
	}

	Object.defineProperty(childern,'getById',{value:getById,writable:false,configurable:false,enumerable:false});
	
	//Once the definition is complete you must return the object
	return children;
}
```

##Creating a Datastructure
Datastructures are a collection of Datagrams and/or Datasets in any hierarchy.  Children are added to datagrams by adding to the fieldMap property.  Children are added to Datasets by setting the items and key properties. 

For example (using the previously defined Datasets and Datagrams:
```javascript
var parentWithKids = new Parent();
parentWithKids.fieldMap.kids = new Children();
parentWithKids.fieldMap.kids.key = 'childID';
parentWithKids.fieldMap.kids.items = function () {
	var child = new Child();
	return child;
};
```
As you can see the fieldMap for the Parent Datagram is extended by adding a new Children datagram to the field map as kids.  The Children datagram is extended by setting the key which is the name of the field that will be used to identify unique rows. For each unique key value a new element (defined by items) will be created.  Items is a function that returns the type of datagram to be created.
You can use multiple Datastructures to describe the same data.

For instance:
```javascript
var kidsWithParent = new Childern();
kidsWithParent.key = 'childID';
kidsWithParent.items= function() {
	var child = new Child();
	child.fieldMap.parent = new Parent();
	return child;
};
```
##Applying Data
Once a Datastructure has been created you can add data to the objects by invoking the fill method supplying an array of structures.

For example the following data may have come from the following SQL Query: 
```SQL
Select * from Parents left Join Children on Parents.mainID = Children.mainID where mainID = 123
```
Returns the data object below:
```javascript
data = [
	{mainID:123
	mainName:'Parent Name',
	mainProperty:'Taller than children',
	childID:1,
	childName:'Billie',
    childProperty:'Fast',
	childType:'Good'},
	{mainID:123
	mainName:'Parent Name',
	mainProperty:'Taller than children',
	childID:2,
	childName:'Jimmy',
    childProperty:'Slow',
	childType:'Bad'},
]
```
To populate the Datadtructure call the fill functon on the top object

```javascript
parentWithKids.fill(data);
```
This will result in the following object
```javascript
{id:123,
name:'Parent Name',
property1:'Taller than children',
kids:[
	{id:1, name:'Billie',property1:'Fast',type:'Good'},
	{id:2, name:'Jimmy', property1:'Slow',type:'Bad'}
]
```
Additionally, you can call the methods you have added to the Datasets and Datastructure for example

kidsWithParents.kids[0].isFast() will return true.

See the test file for more examples.

#Extra stuff
###Debug
**enableDebug()** - turns on verbose output for diagnosing issues, output is sent to console.

###Datagram
**has(key, type)** - returns true if the the key contains the type. For example kidsWithParents.has('kids','Children') returns true

###Dataset
**has(type)** - returns true if the dataset is populated by the requested type.  For example  kidsWithParents.kids.has('Child') returns true.

**copyReturn(sourceArray, destinationDataset)** - returns the source array as the destination dataset.

Questions, comments, concerns please let me know.
