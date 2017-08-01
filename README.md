### epJSON Current status

Still targeting 8.8 release

#### Finished


* IDF parser (in standalone class)

* IDD parser (python script)

* IDF to epJSON translator (in standalone class)

* IDD to epJSON Schema translator (part of build process, python script)

* epJSON Schema validation (in standalone class)

* Change extensions (extensible fields) to appropriately named keys (part of build process)

* Clean up IDD inconsistencies

* Error reporting for IDF parsing, epJSON parsing, and epJSON schema validation

* Refactor InputProcessor into class

* Refactor existing IP APIs to use JSON data structure (critically important for compability with existing code)

* Create new IP APIs for new OOP classes to directly use JSON data structure

* Added a global factory that stores ALL constructed objects that have been converted to OOP

* Refactor GLHEVert and GLHESlinky to use JSON data structure directly

* Refactored code to avoid O(n^2) algorithm in GetObjectItem and VerifyName

* Moved functions like FindItemInList, FindItem, VerifyName, etc to UtilityRoutines

* Remove CacheIPError file

* Embed binary CBOR epJSON Schema into compiled executable (no need to read schema file at runtime)

* Command line argument (-c, --convert) to output epJSON if using IDF input and IDF if using epJSON input

#### ToDo


* Provide better error messaging when using IDF input (specific errors in IDF format)

* Add unit tests to ensure complete coverage of JSON Schema validation

* General code cleanup

### Command line

IDF input (with epJSON converted output)

`./energyplus -c -d Outputs/ -w USA_IL_Chicago-OHare.Intl.AP.725300_TMY3.epw 1ZoneUncontrolled.idf`

epJSON input (with IDF converted output)

`./energyplus -c -d Outputs/ -w USA_IL_Chicago-OHare.Intl.AP.725300_TMY3.epw 1ZoneUncontrolled.epJSON`


### Global factory

```cpp
GLHEVert::GLHEVert( std::string const & name, json const & fields )
	{
		bool errorsFound = false;
		std::string const cCurrentModuleObject = "GroundHeatExchanger:Vertical";

		Name = name;
		UtilityRoutines::IsNameEmpty( Name, cCurrentModuleObject, errorsFound );

		auto const inletNodeName = UtilityRoutines::MakeUPPERCase( fields.at( "inlet_node_name" ) );
		auto const outletNodeName = UtilityRoutines::MakeUPPERCase( fields.at( "outlet_node_name" ) );

		//get inlet node num
		inletNodeNum = NodeInputManager::GetOnlySingleNode( inletNodeName, errorsFound, cCurrentModuleObject, Name, NodeType_Water, NodeConnectionType_Inlet, 1, ObjectIsNotParent );
		//get outlet node num
		outletNodeNum = NodeInputManager::GetOnlySingleNode( outletNodeName, errorsFound, cCurrentModuleObject, Name, NodeType_Water, NodeConnectionType_Inlet, 1, ObjectIsNotParent );

		available = true;
		on = true;

		BranchNodeConnections::TestCompSet( cCurrentModuleObject, Name, inletNodeName, outletNodeName, "Condenser Water Nodes" );

		//load borehole data
		designFlow = fields.at( "design_flow_rate" );
		PlantUtilities::RegisterPlantCompDesignFlow( inletNodeNum, designFlow );

		numBoreholes = fields.at( "number_of_bore_holes" );
		boreholeLength = fields.at( "bore_hole_length" );
		boreholeRadius = fields.at( "bore_hole_radius" );
		kGround = fields.at( "ground_thermal_conductivity" );
		cpRhoGround = fields.at( "ground_thermal_heat_capacity" );
		tempGround = fields.at( "ground_temperature" );
		kGrout = fields.at( "grout_thermal_conductivity" );
    ...
  }
```

Example showing `objectFactory()` usage
```cpp
} else if ( UtilityRoutines::SameString( this_comp_type, "GroundHeatExchanger:Vertical" ) ) {
  this_comp.TypeOf_Num = TypeOf_GrndHtExchgVertical;
  this_comp.GeneralEquipType = GenEquipTypes_GroundHeatExchanger;
  this_comp.CurOpSchemeType = UncontrolledOpSchemeType;
  this_comp.compPtr = inputProcessor->objectFactory< GroundHeatExchangers::GLHEVert >( CompNames( CompNum ) );
} else if ( UtilityRoutines::SameString( this_comp_type, "GroundHeatExchanger:Surface" ) ) {
```

All new objects must implement `canonicalObjectType()` and `objectTypeHash()`
```cpp
std::string const & GLHEVert::canonicalObjectType() {
  static const std::string objectType = "GroundHeatExchanger:Vertical";
  return objectType;
}

std::size_t GLHEVert::objectTypeHash() {
  static const std::size_t objectTypeHash = std::hash< std::string>{}( canonicalObjectType() );
  return objectTypeHash;
}
```

The data "storage" for all constructed classes
```cpp
	using deleter_t = std::function< void( void * ) >;
	using unique_void_ptr = std::unique_ptr< void, deleter_t >;

	template< typename T >
	auto unique_void( T * ptr ) -> unique_void_ptr
	{
		auto deleter = []( void * data ) -> void {
			T * p = static_cast< T * >( data );
			delete p;
		};
		return std::unique_ptr< void, deleter_t >( ptr, deleter );
	}

	std::unordered_map< std::size_t, std::unordered_map< std::string, unique_void_ptr > > storage;
```

How to add new objects to data storage
```cpp
template< typename T >
T *
addObject( std::string const & name, json const & fields )
{
  T * ptr = new T( name, fields );
  storage[ T::objectTypeHash() ][ name ] = std::move( unique_void( ptr ) );
  return ptr;
}

template< typename T >
T *
addObject( json const & fields )
{
  static const std::string blankString;
  T * ptr = new T( fields );
  storage[ T::objectTypeHash() ][ blankString ] = std::move( unique_void( ptr ) );
  return ptr;
}

template< typename T >
std::vector< T * >
addObjects( json const & objs )
{
  assert( objs.is_object() );
  std::vector< T * > output;
  output.reserve( objs.size() );
  for ( auto it = objs.begin(); it != objs.end(); ++it ) {
    T * ptr = new T( it.key(), it.value() );
    output.emplace_back( ptr );
    storage[ T::objectTypeHash() ][ it.key() ] = std::move( unique_void( ptr ) );
  }
  return output;
}
```

The generic factory methods to get a pointer to constructed and stored objects
```cpp
template< typename T >
T *
objectFactory( std::string const & name )
{
  auto const it = storage.find( T::objectTypeHash() );
  if ( it == storage.end() ) return nullptr;
  auto const it2 = it->second.find( name );
  if ( it2 == it->second.end() ) return nullptr;
  T * p = static_cast< T * >( it2->second.get() );
  return p;
}

template< typename T >
T *
objectFactory()
{
  static const std::string blankString;
  auto const it = storage.find( T::objectTypeHash() );
  if ( it == storage.end() ) return nullptr;
  auto const it2 = it->second.find( blankString );
  if ( it2 == it->second.end() ) return nullptr;
  T * p = static_cast< T * >( it2->second.get() );
  return p;
}
```

Public `objectFactory()` called from InputProcessor

This will try to find a constructed object otherwise add new object
```cpp
template< typename T >
T *
objectFactory( std::string const & objectName )
{
  T * p = data->objectFactory< T >( objectName );
  if ( p != nullptr ) return p;
  auto const & fields = getFields( T::canonicalObjectType(), objectName );
  p = data->addObject< T >( objectName, fields );
  return p;
}

template< typename T >
T *
objectFactory()
{
  T * p = data->objectFactory< T >();
  if ( p != nullptr ) return p;
  auto const & fields = getFields( T::canonicalObjectType() );
  p = data->addObject< T >( fields );
  return p;
}
```

Example python script to modify epJSON file
```python
import json
import os

with open(os.path.join(os.path.dirname(__file__), "1ZoneUncontrolled.epJSON")) as f:
    input_file = json.load(f)

sun_exposure = input_file['BuildingSurface:Detailed']['Zn001:Flr001']['sun_exposure']
print(sun_exposure)

input_file['BuildingSurface:Detailed']['Zn001:Flr001']['sun_exposure'] = 'SunExposed'

with open(os.path.join(os.path.dirname(__file__), "1ZoneUncontrolled.epJSON")) as f:
    json.dump(input_file, f, sort_keys=True, indent=4)
```

Example python script to validate epJSON against schema
```python
import json
import jsonschema
import os

with open(os.path.join(os.path.dirname(__file__), "1ZoneUncontrolled.epJSON")) as f2:
    input_file = json.load(f2)

with open(os.path.join(os.path.dirname(__file__), "Energy+.schema.epJSON")) as f:
    schema = json.load(f)

jsonschema.validate(input_file, schema)
```

Example epJSON
```
{
  "BuildingSurface:Detailed": {
    "Zn001:Flr001": {
        "construction_name": "FLOOR",
        "number_of_vertices": 4,
        "outside_boundary_condition": "Adiabatic",
        "outside_boundary_condition_object": "",
        "sun_exposure": "NoSun",
        "surface_type": "Floor",
        "vertices": [
            {
                "vertex_x_coordinate": 15.24,
                "vertex_y_coordinate": 0.0,
                "vertex_z_coordinate": 0.0
            },
            {
                "vertex_x_coordinate": 0.0,
                "vertex_y_coordinate": 0.0,
                "vertex_z_coordinate": 0.0
            },
            {
                "vertex_x_coordinate": 0.0,
                "vertex_y_coordinate": 15.24,
                "vertex_z_coordinate": 0.0
            },
            {
                "vertex_x_coordinate": 15.24,
                "vertex_y_coordinate": 15.24,
                "vertex_z_coordinate": 0.0
            }
        ],
        "view_factor_to_ground": 1,
        "wind_exposure": "NoWind",
        "zone_name": "ZONE ONE"
    }
}
```

Comparable IDF
```
BuildingSurface:Detailed,
  Zn001:Flr001,            !- Name
  Floor,                   !- Surface Type
  FLOOR,                   !- Construction Name
  ZONE ONE,                !- Zone Name
  Adiabatic,               !- Outside Boundary Condition
  ,                        !- Outside Boundary Condition Object
  NoSun,                   !- Sun Exposure
  NoWind,                  !- Wind Exposure
  1.000000,                !- View Factor to Ground
  4,                       !- Number of Vertices
  15.24000,0.000000,0.0,  !- X,Y,Z ==> Vertex 1 {m}
  0.000000,0.000000,0.0,  !- X,Y,Z ==> Vertex 2 {m}
  0.000000,15.24000,0.0,  !- X,Y,Z ==> Vertex 3 {m}
  15.24000,15.24000,0.0;  !- X,Y,Z ==> Vertex 4 {m}
```

#### JSON Outputs

* JSON outputs for Tabular and Timeseries outputs

* Outputs separate files (as part of try/catch/finally) for detailed_zone/detailed_HVAC/timestep/hourly/daily/monthly/runperiod

* Can output CBOR and MessagePack binary formats

* Intended for 8.8 release (dependent on JSON input)

* Eventually it could replace ESO, CSV, etc but that is open to discussion

```
Output:JSON,
       \memo Output from EnergyPlus can be written to JSON format files.
       \unique-object
  A1 , \field Option Type
  	   \required-field
       \type choice
       \key TimeSeries
       \key TimeSeriesAndTabular
  A2 , \field Output JSON
       \type choice
       \key Yes
       \key No
       \default Yes
  A3 , \field Output CBOR
       \type choice
       \key Yes
       \key No
       \default No
  A4 ; \field Output MessagePack
       \type choice
       \key Yes
       \key No
       \default No
```

```
Running the Outpatient Reference Model for an annual simulation has the following file sizes:

eplusout.mtr - 2.1M
eplusmtr.csv - 1.6M
eplusout.eso - 31M
eplusout.csv - 24M
eplusout.sql - 31M

eplusout_hourly.json - 42M
eplusout_hourly.cbor - 12M (~71% reduction to JSON, 50% compared to CSV)

eplusout.json - 5.0M
eplusout.cbor - 1.2M (76% reduction)
```

#### General thoughts

* epJSON input will be used internally and as experimental input in 8.8 (to allow for schema changes and user input around key names)

* In EnergyPlus 8.9, epJSON becomes 1st class citizen along with IDF.

* In EnergyPlus 9.0 or 9.1, deprecate IDF input, but still have automatic translation within E+

* In future version of Energyplus, finally remove IDF input, freeze IDD/IDF, and move translation program out of E+

* This should minimize impact on existing IDF workflows and provide a long time for users/companies to transition.
