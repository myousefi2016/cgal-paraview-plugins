## Creating a CGAL Plugin for Paraview

This page is a tutorial detailing the creation of a CGAL plugin for the data analysis and visualization software [Paraview](https://www.paraview.org/). The different steps of the process are illustrated using code from a CGAL plugin that can run CGAL's [isotropic remeshing](https://doc.cgal.org/latest/Polygon_mesh_processing/index.html#title7) algorithm on a triangle mesh given as a `vtkPolyData` object. This repository contains all the necessary source files related to the example plugin such that you can compile, run, and tweak it as you desire.

<p align="center">
<img src="https://github.com/CGAL/cgal-web/blob/6d0e5cc6182b2e1ca42ed13599b902a4a5a0941e/images/Isotropic_remeshing_2.png" alt="drawing" width="750"/>
</p>

> **_NOTE:_** **Since a Paraview plugin can only be loaded within the same version of Paraview that it was built with, a developer version of Paraview is required. Furthermore, this example plugin and tutorial is restricted to versions 5.6 or earlier of Paraview due to a change of API.**

### Building Paraview from Sources

First, obtain Paraview's sources from [the official repository](https://gitlab.kitware.com/paraview/paraview). You can either directly download the release 5.6, or clone the repository and check out the tag `5.6`. Note that if you chose to clone the repository, you will also need to update the submodules. To do so, go to the source directory, and run:

```
git submodule update --init --recursive
```

You can now create a build directory, and build Paraview:
```
mkdir <path-to-your-build-directory>
cd <path-to-your-build-directory>
cmake <path-to-Paraview-sources>
make
```

### Writing a Plugin

Now that you have built Paraview, and assuming you already have a ready-to-use version of CGAL (see [this helper page](https://doc.cgal.org/latest/Manual/installation.html) otherwise), you are ready to start writing your plugin.

A few files are expected to make a functional plugin, and this comes from the fact that [VTK](https://vtk.org/) (the underlying software beneath Paraview) is based on a pipeline that we can roughly describe as follows: a *source* creates a VTK object, which can then serve as input for a *filter*, which performs its algorithm when its `Update()` function is called. A Paraview *plugin* is the association between a filter of this pipeline and an XML file used to create a UI element.

Most of the time, we want to interact with VTK data structures. In the case of our isotropic remeshing plugin, our input data is a `vtkPolyData`, and so we derive our filter class from the class [`vtkGeometryFilter`](https://vtk.org/doc/release/5.6/html/a00712.html).

In our case, the filter must implement the following methods to interact with the pipeline:
- `RequestDataObject()`
   This function is used to create the output object. For our plugin, this is where the `vtkPolyData` object that will hold the remeshed data is constructed,
   and it is not needed because the class `vtkGeometryFilter` already implements it (it returns an empty `vtkPolyData`).
- `RequestInformation()`
   In this function, we do all possible light-weight computations, such as computing the bounding box of the input data, for example. This function is called before `Update()` and provides information that might be needed from the input. In our case, this is where we compute the default target edge length.
   Note: In some algorithms, this function is replaced by `ExecuteInformation()`. Before VTK 5, `Executeinformation()` was the standard, and some algorithms used for the transition have kept it this way, so you should check the documentation of the base class that you have chosen before you write this function, as it might actually never be called.
- `RequestData()`
   This is where the input is acquired, the main algorithm is performed, and the output is filled. In our case, we run CGAL's isotropic remeshing algorithm.
- `FillInputPortInformation()`
   This is where we specify the type of the object that we expect as input.
- `FillOutputPortInformation()`
   This is where we specify the type of the output object.

#### Header File
```c++
#ifndef vtkIsotropicRemeshingFilter_h
#define vtkIsotropicRemeshingFilter_h
// Gives access to macros for communication with the UI
#include "vtkFiltersCoreModule.h" 
#include "vtkGeometryFilter.h"

// Inherit from the desired filter
class vtkIsotropicRemeshingFilter : public vtkGeometryFilter
{
public:
  // VTK requirements
  static vtkIsotropicRemeshingFilter* New();
  vtkTypeMacro(vtkIsotropicRemeshingFilter, vtkGeometryFilter);
  // Prints the values of the specific data
  void PrintSelf(ostream& os, vtkIndent indent) override;

  // Communicate with the UI
  vtkSetMacro(Length, double);
  vtkGetMacro(Length, double);
  vtkSetMacro(LengthInfo, double);
  vtkGetMacro(LengthInfo, double);
  vtkSetMacro(MainIterations, int);
  vtkGetMacro(MainIterations, int);

  // Pipeline functions:
  // Performs the isotropic remeshing algorithm and fills the output object here.
  int RequestData(vtkInformation *, vtkInformationVector **, vtkInformationVector *)override;
  // Specifies the type of the input objects
  int FillInputPortInformation(int, vtkInformation *info)override;
  // Specifies the type of the output object.
  int FillOutputPortInformation(int, vtkInformation *info)override;

protected:
  vtkIsotropicRemeshingFilter();
  ~vtkIsotropicRemeshingFilter(){}

  // Computes the bbox's diagonal length to set the default target edge length.
  int RequestInformation(vtkInformation *, vtkInformationVector **, vtkInformationVector *);

private:
  // Data set by the UI and used by the algorithm
  double Length;
  double LengthInfo;
  int MainIterations;
  
  // needed but not implemented
  vtkIsotropicRemeshingFilter(const vtkIsotropicRemeshingFilter&);
  void operator=(const vtkIsotropicRemeshingFilter&);
};
#endif

```
#### Source File

```c++
#include <ostream>
#include <sstream>
#include "vtkInformationVector.h"
#include "vtkIsotropicRemeshingFilter.h"
#include "vtkPolyData.h"
#include "vtkInformation.h"
#include <CGAL/Surface_mesh.h>
#include <CGAL/Simple_cartesian.h>
#include <CGAL/Polygon_mesh_processing/remesh.h>

// Declare the plugin
vtkStandardNewMacro(vtkIsotropicRemeshingFilter);

namespace PMP = CGAL::Polygon_mesh_processing;
// Useful typedefs
typedef CGAL::Simple_cartesian<double>                            K;
typedef CGAL::Surface_mesh<K::Point_3>                            SM;
typedef boost::property_map<SM, CGAL::vertex_point_t>::type       VPMap;
typedef boost::property_map_value<SM, CGAL::vertex_point_t>::type Point_3;
typedef boost::graph_traits<SM>::vertex_descriptor                vertex_descriptor;
typedef boost::graph_traits<SM>::edge_descriptor                  edge_descriptor;
typedef boost::graph_traits<SM>::face_descriptor                  face_descriptor;
typedef boost::graph_traits<SM>::halfedge_descriptor              halfedge_descriptor;

// -----------------------------------------------------------------------------
// Constructor
// Fills the number of input and output objects.
// Initializes the members that need it.
vtkIsotropicRemeshingFilter::vtkIsotropicRemeshingFilter()
{
  SetNumberOfInputPorts(1);
  SetNumberOfOutputPorts(1);
}

// ----------------------------------------------------------------------------
// Gets the input
// Creates CGAL::Surface_mesh from vtkPolydata
// Calls the CGAL::isotropic_remeshing algorithm
// Fills the output vtkPolyData from the result.
int vtkIsotropicRemeshingFilter::RequestData(vtkInformation *,
                                             vtkInformationVector **inputVector,
                                             vtkInformationVector *outputVector)
{
  //  Get the input and output data objects.
  //  Get the info objects
  vtkInformation *inInfo = inputVector[0]->GetInformationObject(0);
  vtkInformation *outInfo = outputVector->GetInformationObject(0);
  //  Get the input
  vtkPolyData *polydata = vtkPolyData::SafeDownCast(inInfo->Get(vtkDataObject::DATA_OBJECT()));
  
  /********************************************
   * Create a SurfaceMesh from the input mesh *
   ********************************************/
  SM sm;
  VPMap vpmap = get(CGAL::vertex_point, sm);
  
  //  Get nb of points and cells
  vtkIdType nb_points = polydata->GetNumberOfPoints();
  vtkIdType nb_cells = polydata->GetNumberOfCells();
  
  // Extract points
  std::vector<vertex_descriptor> vertex_map(nb_points);
  for (vtkIdType i=0; i<nb_points; ++i)
  {
    double coords[3];
    polydata->GetPoint(i, coords);
    vertex_descriptor v = add_vertex(sm);
    put(vpmap, v, K::Point_3(coords[0], coords[1], coords[2]));
    vertex_map[i] = v;
  }
  
  // Extract cells
  for (vtkIdType i = 0; i<nb_cells; ++i)
  {
    vtkCell* cell_ptr = polydata->GetCell(i);
    vtkIdType nb_vertices = cell_ptr->GetNumberOfPoints();
    std::vector<vertex_descriptor> vr(nb_vertices);
    for (vtkIdType k=0; k<nb_vertices; ++k)
      vr[k] = vertex_map[cell_ptr->GetPointId(k)];
    CGAL::Euler::add_face(vr, sm);
  }
  
  std::vector<vertex_descriptor> isolated_vertices;
  for(SM::vertex_iterator vit = sm.vertices_begin();
      vit != sm.vertices_end();
      ++vit)
  {
    if(sm.is_isolated(*vit))
      isolated_vertices.push_back(*vit);
  }
  
  for (std::size_t i=0; i < isolated_vertices.size(); ++i)
    sm.remove_vertex(isolated_vertices[i]);

  if(!is_triangle_mesh(sm))
  {
    vtkErrorMacro("The input mesh must be triangulated ");
    return 0;
  }
  
  /*****************************
   * Apply Isotropic remeshing *
   *****************************/
  PMP::isotropic_remeshing(sm.faces(),
                           Length,
                           sm,
                           PMP::parameters::number_of_iterations(MainIterations));
  
  /**********************************
   * Pass the SM data to the output *
   **********************************/
  vtkPolyData *output = vtkPolyData::SafeDownCast(outInfo->Get(vtkDataObject::DATA_OBJECT()));
  vtkNew<vtkPoints> const vtk_points;
  vtkNew<vtkCellArray> const vtk_cells;
  vtk_points->Allocate(sm.number_of_vertices());
  vtk_cells->Allocate(sm.number_of_faces());
  std::vector<vtkIdType> Vids(sm.number_of_vertices());
  vtkIdType inum = 0;
  
  for(vertex_descriptor v : vertices(sm))
  {
    const K::Point_3& p = get(vpmap, v);
    vtk_points->InsertNextPoint(CGAL::to_double(p.x()),
                                CGAL::to_double(p.y()),
                                CGAL::to_double(p.z()));
    Vids[v] = inum++;
  }
  
  for(face_descriptor f : faces(sm))
  {
    vtkNew<vtkIdList> cell;
    for(halfedge_descriptor h : halfedges_around_face(halfedge(f, sm), sm))
      cell->InsertNextId(Vids[target(h, sm)]);

    vtk_cells->InsertNextCell(cell);
  }
  
  output->SetPoints(vtk_points);
  output->SetPolys(vtk_cells);
  output->Squeeze();

  return 1;
}

// ----------------------------------------------------------------------------
void vtkIsotropicRemeshingFilter::PrintSelf(std::ostream& os, 
                                            vtkIndent indent)
{
  this->Superclass::PrintSelf(os,indent);
  os<<"Length        : "<<Length        <<std::endl;
  os<<"LengthInfo    : "<<LengthInfo    <<std::endl;
  os<<"MainIterations: "<<MainIterations<<std::endl;
}

// ------------------------------------------------------------------------------
int vtkIsotropicRemeshingFilter::FillInputPortInformation(int vtkNotUsed(port),
                                                          vtkInformation* info)
{
  info->Set(vtkAlgorithm::INPUT_REQUIRED_DATA_TYPE(), "vtkPolyData");
  return 1;
}

// ------------------------------------------------------------------------------
int vtkIsotropicRemeshingFilter::FillOutputPortInformation(int,
                                                           vtkInformation *info)
{
  // Always returns a vtkPolyData
  info->Set(vtkDataObject::DATA_TYPE_NAME(), "vtkPolyData");
  return 1;
}

// ------------------------------------------------------------------------------
int vtkIsotropicRemeshingFilter::RequestInformation(vtkInformation *,
                                                    vtkInformationVector ** inputVector,
                                                    vtkInformationVector *outputVector)
{
  vtkInformation *inInfo = inputVector[0]->GetInformationObject(0);
  vtkInformation *outInfo = outputVector->GetInformationObject(0);
  
  // Sets the bounds of the output.
  outInfo->Set(vtkDataObject::BOUNDING_BOX(),
               inInfo->Get(vtkDataObject::BOUNDING_BOX()),
               6);

  vtkPolyData *input= vtkPolyData::SafeDownCast(
        inInfo->Get(vtkDataObject::DATA_OBJECT()));
  
  // Computes the initial target length:
  double * bounds = input->GetBounds();
  double diagonal = std::sqrt((bounds[0]-bounds[1]) * (bounds[0]-bounds[1]) +
                              (bounds[2]-bounds[3]) * (bounds[2]-bounds[3]) +
                              (bounds[4]-bounds[5]) * (bounds[4]-bounds[5]));
  SetLengthInfo(0.01*diagonal);
  
    return 1;
}

```

#### XML
The XML file is used to configure the widget in the UI. This is where we define the `LineEdit` to specify input parameters, such as the target edge length and the number of iterations in our case.

```xml
<ServerManagerConfiguration>
  <ProxyGroup name="filters">
    <SourceProxy name="IsotropicRemeshingFilter" class="vtkIsotropicRemeshingFilter" label="Isotropic Remeshing">
    <Documentation
      short_help="Remeshes datasets."
      long_help="Remeshes a dataset.">
      This filter will remesh the given data set. It takes a PolyData 
      as input, and returns a PolyData containing the remeshed input.
    </Documentation>
<!-- Dialog to choose the input -->
      <InputProperty
        name="Mesh"
        port_index="0"
        command="SetInputConnection">
        <ProxyGroupDomain name="groups">
          <Group name="sources"/>
          <Group name="filters"/>
        </ProxyGroupDomain>
        <DataTypeDomain name="input_type">
          <DataType value="vtkDataSet"/>
        </DataTypeDomain>
        <Documentation>
          The input Meshes.
        </Documentation>
      </InputProperty>
<!-- Elements of configuration of the filter -->
      <DoubleVectorProperty name="Length"
        command="SetLength"
        label="Target Edge Length"
        number_of_elements="1"
        default_values="0"
        information_property="LengthInfo"
        >
         <DoubleRangeDomain name="range" min="0.0" />
        <Documentation>
          The target length for the new edges of the mesh (default is 1% of the bounding box).
        </Documentation>
      </DoubleVectorProperty>
      <DoubleVectorProperty name="LengthInfo"
                            command="GetLengthInfo"
                            information_only="1">
        <SimpleDoubleInformationHelper />
      </DoubleVectorProperty>
      <IntVectorProperty name="MainIterations"
        command="SetMainIterations"
        label="#Main Iterations"
        number_of_elements="1"
        default_values="1">
        <Documentation>
          The number of iterations for the sequence of atomic operations performed (edge splits,
          edge collapses, edge flips, tangential relaxation and projection to the initial surface
          to generate a smooth mesh with a prescribed edge length).
        </Documentation>
      </IntVectorProperty>
      <!-- Show in the Filters menu under "CGAL"-->
      <Hints>
        <ShowInMenu category="CGAL" />
      </Hints>
    </SourceProxy>
  </ProxyGroup>
</ServerManagerConfiguration>

```

#### CMakeLists

On the CMake side, the most important thing is the `ADD_PARAVIEW_PLUGIN` macro. It takes the sources and the XML file of the plugin as arguments, and deals with most of the VTK parts.

```cmake
cmake_minimum_required(VERSION 3.1 FATAL_ERROR)
project(CGAL_Isotropic_remeshing_filter)
#If the plugin is used internally (inside Paraview's source directory),
#then we don't need to call find_package.
if (NOT ParaView_BINARY_DIR)
  find_package(ParaView REQUIRED)
endif()
#Find CGAL
find_package(CGAL)
if(CGAL_FOUND)
  include( ${CGAL_USE_FILE} )
endif(CGAL_FOUND)
if(ParaView_FOUND)
  #If the plugin is used internally we don't need to include.
  if (NOT ParaView_BINARY_DIR)
      include(${PARAVIEW_USE_FILE})
  endif(ParaView_BINARY_DIR)
  
  #Paraview's macro to add the plugin. It takes care of all the vtk 
  #and paraview parts of the process, like link and integration
  #in the UI
  ADD_PARAVIEW_PLUGIN(IsotropicRemeshingFilter "1.0"
    SERVER_MANAGER_XML IsotropicRemeshingFilter.xml
    SERVER_MANAGER_SOURCES vtkIsotropicRemeshingFilter.cxx)
else()
  message("WARNING : The Paraview plugins need Paraview, so they won't be compiled.")
endif(ParaView_FOUND)
# Link with CGAL
if(CGAL_FOUND)
  if(ParaView_FOUND)
    target_link_libraries(IsotropicRemeshingFilter LINK_PRIVATE
      CGAL::CGAL 
      ${Boost_LIBRARIES})
  endif(ParaView_FOUND)
endif (CGAL_FOUND)
```

### Building the Plugin

To build a plugin, use CMake and provide the location of the CGAL library with `CGAL_DIR`. For example:
```
  cd <path-to-build-dir>
  cmake -DCGAL_DIR=~/CGAL/releases/CGAL-4.9/cmake/platforms/release -DParaView_DIR=<PATH-TO-Paraview-DIR> <path-to-dir-containing-plugin's-CMakeLists.txt>
  make
```

Until Paraview 5.6.0 (included), the variable `Paraview_DIR` is simply the build directory, where the file `ParaViewConfig.cmake` can be found. 

> **_NOTE:_** **Be careful to use the same version of Qt when compiling the plugin and when compiling Paraview.**

### Loading the Plugin in Paraview

Launch Paraview, go to `Tools->Manage Plugins` and click on `Load New`. Select the library file of your plugin in the list, and click `Close`. The plugin should appear in the `Filter List`. Your plugin is now ready to use!

### Further Information

You can find further information on VTK and Paraview on their [wiki](https://vtk.org/Wiki/VTK/) and [documentation](https://vtk.org/documentation/) pages. For CGAL questions or issues, you can use the channels described on [this page](https://www.cgal.org/mailing_list.html) as well as the issues of this Github repository.
