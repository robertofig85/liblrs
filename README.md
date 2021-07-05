# ..:: LibLRS ::..

## Introduction

Library for performing linear referencing workflow on geographic data, written in C++.

A Linear Referencing System (LRS) allows one to query locations and events based on their distance along a line, instead of their spatial coordinates. It is often used with transportation data, such as identifying a point located x distance from a known location.

## How to build

This library can be built into a DLL by using the "build.bat" file, or included directly in a C++ project. This library uses GDAL as a depencency, so having it installed is mandatory (version > 2.0 guaranteed, probably works on earlier as well). If building into a DLL, you must edit the include and lib paths to point to your own install of GDAL. A debug build can be generated by calling "build.bat -debug".

## How to use

Linear referencing works with measured lines. This means that each vertex has an M coordinate value, representing the cumulative distance from that vertex to the begining of the linestring. Some of the functions in this library require a previously measured line, while others are used to create one, and thus only require simple linestrings.

If built separately, include "liblrs.h", otherwise include "liblrs.cpp". The header file specifies the namespace 'liblrs' for its structs and typedefs. Most of the structs are used internally, the ones of interest for using the library are 'vec2' and 'vec3', located in "liblrs-spatial.h" and that provide vector math for handling points in space. Most of the functions work on GDAL types, and are meant to be called during a typical loop through the features of a data layer, such as

```c
OGRFeatureH Feature;
while ((Feature = OGR_L_GetNextFeature(Layer)) != NULL)
{
    // call functions here.
}
```

## Functions

```c
vec3 LocatePointFromMeasurement(
    OGRGeometryH Geom,
    double Measurement
);
```
Given a measured linestring and a distance value along it, return the geographic location of at that distance.
* In: [Geom], a LinestringM geometry previously measured; [Measurement], a value between the M values of the first and last vertices.
* Out: a vec3 point at that measurement, or a NaN point in case the measurement is outside the scope of the line.

```c
double GetMeasurementAtPoint(
    OGRGeometryH Geom,
    vec3 Point
);
```
Given a measured linestring an a point, returns the measurement at that point. The function makes no check whether the point is located on top of the line or not; in case not, the measurement at the closest point in the line is returned.
* In: [Geom], a LinestringM geometry previously measured; [Point], the coordinate of the location to be measured.
* Out: a measurement value of the closest point in the line.

```c
interval_list PrepareRasters(
    OGRGeometryH InGeom,
    OGRLayerH MDEBoundaries
);
```
When measuring a line, the user should call this function first if they have height information in the form of a Digital Terrain Model that they want to use when calculating the distances. This functions queries any raster that intersects with the line, and loads it to be used in the measuring function. Since a line may only partially intersect with a raster (instead of being entirely contained in it), it is created a list of sections of the line, separating which ones will receive height information and which won't. Furthermore, several rasters may intersect the line, in which case the function takes all of them into consideration, prioritising higher resolution rasters in case of overlap.
* In: [InGeom], a LinestringZ geometry, the Z data can be blank; [MDEBoundaries], a vector layer of Polygon geometry with the precise boundaries of the rasters to be taken into consideration. The first field must be an Id field, and the second a string with the path for the image; other fields are ignored.
* Out: a list with intervals and raster information to be passed to the measuring function. If no raster intersects the line, the return is a single interval from start to finish that will be measured in 2D.

```c
OGRGeometryH MeasureLine(
    OGRGeometry InGeom,
    interval_list* ListPtr,
    calc_info Calc
);
```
Calculates the cumulative distance from the vertices of a line to its origin, and assigns these values to the M coordinate of each respective vertex. There are several methods for calculating distance available, the preferred method must be passed as a callback in the calc_info struct. The ./utils folder in this project contains a benchmark for the different methods, evaluating accuracy and speed. Furthermore, if there are height rasters onto which to drape the line, this can be passed in the interval_list that is returned from the PrepareRasters() function. If passed, for each line, whereever there is height information it will consider that in the calculations (if the distance method chosen supports 3D measurements). Otherwise the user can pass a NULL value.
* In: [InGeom], a LinestringZ geometry to be measured, the Z value can be blank; [ListPtr], a pointer to the returned struct from PrepareRasters, or NULL; [Calc], a struct with callback functions for distance calculation. The member Calc.Transform is a callback for a transformation function required in certain methods, or NULL for the ones that don't, and Calc.Distance is the distance function callback. Some possible combinations are present in the ./utils benchmark. The members Calc.Name and Calc.EPSG are reserved for future use.
* Out: a LinestringM measured geometry.
