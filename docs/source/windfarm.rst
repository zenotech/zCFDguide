
Windfarm Modelling
------------------

Overview
^^^^^^^^

Windfarms consist of a set of specific turbine types at geographic locations.  zCFD allows users to specify the turbine characteristics and locations, and will automatically create turbine models within an existing mesh and update the local flow conditions to match the thrust coefficient and tip speed ratio for each turbine.

The locations and TRBX turbine definition
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

zCFD provides a utility function *create_trbx_zcfd_data* that can be called from within the zCFD environment and Python interactive shell. To start the zCFD command line environment, type:

.. code-block:: bash

    > source /INSTALL_LOCATION/zCFD-version/bin/activate

The command prompt will now appear with a (zCFD) prefix.  To run the *create_trbx_zcfd_data* utility, start the interactive Python shell:

.. code-block:: bash

    (zCFD) > python

The utility takes several arguments:

.. code-block:: python

    >>> import zutil.farm as farm
    >>> farm.create_trbx_zcfd_input(\
            case_name='windfarm_name', \
            wind_direction=267.0, \
            reference_wind_speed=10.0, \
            num_processes=48, \
            report_frequency=100, \
            update_frequency=50, \
            reference_point_offset=1.0,\
            model='simple',\'
            turbine_zone_length_factor=1.0,\
            turbine_files=[['xy_file_1.txt','turbine_type_1.trbx'], \
                           ['xy_file_2.txt','turbine_type_2.trbx']])

The utility will read each *[<xy_file> , <trbx_file>]* pair provided.  The *<xy_file>* is a 3-column data file in ASCII format with *<turbine name>, <easting>* and *<northing>* data such as:

.. code-block:: bash

    T001 251865.0 646486.0
    T002 252240.0 646200.0
    ...
    T015 253308.0 647985.0

The TRBX file is a data file with turbine information provided by manufacturers including a description of the turbine geometry and aerodynamic performance characteristics.

The utility extracts key information from the TRBX file to automatically create a turbine definition (*<case_name>_zones.py*) with an entry for each turbine in the array:

.. code-block:: python

    turb_zone = {
        'FZ_1':{
        'type':'disc',
        'def':'./turbine_vtp/T001-237.0.vtp',
        'thrust coefficient':0.821,
        'thrust coefficient curve':[[0.0,0.0],...,[4.0,0.866],...,[51.0,0.0],],
        'tip speed ratio':8.796,
        'tip speed ratio curve':[[0.0,0.0],...,[4.0,7.856],...,[51.0,0.0],],
        'turbine power':3003000.0,
        'turbine power curve':[[0.0,0.0],...,[4.0,167000.0],...,[51.0,0.0],],
        'centre':[251865.0,646486.0,338.5],
        'up':[0.0,0.0,1.0],
        'normal':[-0.8387,-0.5446,0.0],
        'inner radius':1.955,
        'outer radius':60.0,
        'reference plane':'true',
        'reference point':[251764.36,646420.64,338.5],
        'update frequency':50,
        'model': 'simple',
        },
        ...
    }

The *'thrust coefficient'* is defined as a single value, and if a data point curve is present in the TRBX file this data is also supplied as the tuple array *'thrust coefficient curve'*, where the wind speed in metres per second is the index.  If this curve is provided, the utility will interpolate the curve at the reference speed to create the single value. The code will issue a warning if a thrust coefficient greater than 1.0 is specified.

The same approach is taken for the *'tip speed ratio'* single value and curve - which is automatically calculated from the rotor speed array (revolutions per minute) in the TRBX file - and the turbine power curve (see note below regarding model selection).

The *'centre'* is the centre of the disc, which is automatically determined from the nominal hub height in the TRBX file as an offset to the ground height at the specified location.  The local ground height is automatically determined from the VTK output files from a previous solver run.  Note that the VTK ground data can be created with a single cycle of the solver, and does not need to include any turbines.

The vertical orientation is defined by the *'up'* vector - normally this will be the unit vector in the *z*-direction. The *'normal'* defines the vector perpendicular to the disc.  The inner and outer radii are based on the TRBX definition of the size of the disc. No account is made of the hub or tower geometry.

The *'reference point'* defines the location in the flow domain that is used as the reference value of wind velocity for this turbine.  This velocity is used in combination with the thrust coefficient and the tip speed ratio for zCFD to calculate the momentum sources associated with the turbine.  The user specifies the reference point location upstream of the turbine actuator using the keyword *'reference_point_offset'* which is applied as a factor to the rotor diameter.  Thus an offset of 1.0 places the reference point one turbine diameter upstream of the center of the rotor. Also by default a single value is used, but if the *'reference plane'* is set to *'true'* then an averaged value of the turbine zone wind speed in an upstream plane containing the reference point is applied. The flow field is used to update the turbine model every *'update frequency'* timesteps, with a default to every timestep.

The *turbine_vtk/<turbine>.vtp* file defining the fluid zone for each turbine is also automatically created. The diameter of the cylindrical zone matches the turbine outer diameter, and the user specifies the length of the cylinder using the *'turbine_zone_length_factor'*. This factor is automatically multiplied by the turbine diameter.  A warning will be issued and the cylinder length automatically increased if the zone does not include the reference point.  In any case, a warning is issued if the cylinder length is less than one turbine diameter.

The utility also automatically creates a set of monitor points for each turbine, all in a single file (*<case_name>_probes.py*):

.. code-block:: python

    turb_probe = {
                  'report' : {
                               'frequency' : 100,
                               'monitor' : {
                                             'MR_1' : {
                                             'name' :'probe1@MHH@87',
                                             'point' : [251865.0,646486.0,338.5],
                                             'variables' : ['V', 'ti'],
                                                      },
                                             ...
                                           }
                             }
                 }

The *'frequency'* is the number of solver cycles between outputs, and the *'monitor'* defines the name of the probe using the WindFarmer standard notation.

The *'model'* choice depends upon how the user wants zCFD to calculate thrust and torque on the actuator disc. The default *'induction'* model uses linear momentum theory and a specified thrust coefficient based upon the upsteam reference point values (plus an assumption of Betz optimality), whereas *'simple'* uses the specified thrust and turbine power curves to apply the reference values directly to the actuator.  These Python functions are detailed (and can be modified by the user) in:

.. code-block:: bash

    > /INSTALL_LOCATION/zCFD-version/bin/zutil/__init__.py

Because the zone and probe files are automatically created, the following lines must be added to the end of the standard zCFD parameter definition file *<case_name>.py* to insert the data:

.. code-block:: python

    z = zutil.get_zone_info('<case_name>_zones')
    for key,value in z.turb_zone.items():
    parameters[key]=value

    p = zutil.get_zone_info('<case_name>_probes')
    for key,value in p.turb_probe.items():
    parameters[key]=value

When run, zCFD will include the probe data in the *<case_name>_report.csv* file. Note that this utility make take a few seconds to run, especially if there are large numbers of turbines, or points on the mesh boundary.

Writing WindFarmer Data Files
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In order to export the data from a zCFD run in a format that can be read by WindFarmer, we provide the utility *write_windfarmer_data*, with usage:

.. code-block:: python

    >>> import zutil.farm as farm
    >>> farm.write_windfarmer_data(case_name='windfarm_name', \
                                   num_processes=48, \
                                   up = [0,0,1])

The *up* vector is used to check that the orientation expected by WindFarmer is that same as the orientation used in the simulation.  In most cases this will be the *z*-axis.

The utility will output the probe information plus additional fields, calculated automatically.



