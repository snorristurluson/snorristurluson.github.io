---
title: Building a maze
tags: ue4 c++
---
I've been experimenting with the navigation system in Unreal (see my earlier posts,
[Click To Move](/ClickToMove) and [Visualizing Paths](/VisualizingPaths)). What could be the best test
environment for path finding? A maze, maybe? Anyway, making a randomly generated maze seemed like a fun
project so I went ahead and did that.

### Cells
As a first pass, I'm doing a simple and straight forward implementation, generating the maze on a grid
of rectangular cells. Each cell can have four walls (North, East, South, West) and can be marked as
visited.

```cpp
struct FCell
{
  bool Walls[4] = {true, true, true, true};
  bool Visited = false;
};
```
### Generating the maze
When generating the maze, we start with a grid of cells, where each cell has all four walls set, and is
not marked as visited. Note that adjacent cells have two walls between them.

![Maze cells](/images/BuildingAMaze/MazeCells.svg)

The algorithm we use generates a perfect maze - a maze where every cell has exactly one path to get to
every other cell in the maze. Therefore, it doesn't really matter where we start - we'll end up connecting
all the cells anyway. So, let's start at (0,0), in the upper left corner. We mark that cell as visited
and pick a random direction to go in, as long as
* There is a cell in that direction
* It hasn't been visited already.

![First step](/images/BuildingAMaze/FirstStep.svg) ->
![Second step](/images/BuildingAMaze/SecondStep.svg)

We take down the walls between those cells and simply repeat this recursively. If we don't find a cell
that hasn't been visited, we return to the previous invocation. Eventually all cells will have been visited
and we return from the top level function call. At that point we'll have our perfect maze!

![Sample maze](/images/BuildingAMaze/SampleMaze.png)

### The code
The full code is available on GitHub at
[https://github.com/snorristurluson/DungeonCrawler](https://github.com/snorristurluson/DungeonCrawler)
but I'll show the main function here:
```cpp
class FMazeGenerator
{
public:
  FMazeGenerator(int W, int H, int32 Seed);
  ~FMazeGenerator();

  FCell* GetCell(int X, int Y) const;
  FCell* GetNeighbor(int X, int Y, Direction Dir) const;

  void GenerateRecursive(int X, int Y);

protected:
  FCell* Cells;
  int Width;
  int Height;
  FRandomStream RNG;

  int GetNonVisitedNeighbors(int X, int Y, int Neighbors[4]) const;
};
```
The constructor allocates a buffer for the cells and stores a seed for a random stream, allowing us to
regenerate the same maze - very important for testing.

Here is the recursive function for generating the maze:
```cpp
void FMazeGenerator::GenerateRecursive(int X, int Y)
{
  FCell* Cell = GetCell(X, Y);
  Cell->Visited = true;
  UE_LOG(LogTemp, Display, TEXT("Visiting %d, %d"), X, Y);

  while (true)
  {
    int Neighbors[4];
    int NumNeighbors = GetNonVisitedNeighbors(X, Y, Neighbors);

    if (NumNeighbors == 0)
    {
      break;
    }
    int NeighborIndex = 0;
    if (NumNeighbors > 1)
    {
      NeighborIndex = RNG.RandRange(0, NumNeighbors - 1);
    }
    const Direction Dir = static_cast<Direction>(Neighbors[NeighborIndex]);
    FCell* Neighbor = GetNeighbor(X, Y, Dir);
    switch (Dir)
    {
    case North:
      Cell->Walls[North] = false;
      Neighbor->Walls[South] = false;
      GenerateRecursive(X, Y - 1);
      break;
    case East:
      Cell->Walls[East] = false;
      Neighbor->Walls[West] = false;
      GenerateRecursive(X + 1, Y);
      break;
    case South:
      Cell->Walls[South] = false;
      Neighbor->Walls[North] = false;
      GenerateRecursive(X, Y + 1);
      break;
    case West:
      Cell->Walls[West] = false;
      Neighbor->Walls[East] = false;
      GenerateRecursive(X - 1, Y);
      break;
    default:
      break;
    }
  }
}
```

### Rendering the maze
To render the maze, we first need some meshes. I put the following together in Blender:

![Meshes](/images/BuildingAMaze/Meshes.png)

Programmer art at its finest!

I have a Dungeon actor that uses the maze generator to generate the maze, then sets up the meshes to render it.

Each cell is rendered as a combination of the meshes, splitting the cell into a 3x3 grid. For a perfect
maze like we're generating here, the center piece is always a floor mesh. The other pieces depend on the presence
of walls in each direction - the mesh selection is just a long list of if statements, checking all the different
combinations.

Rather than adding an actor to the scene for every mesh, I'm using instanced static meshes. I tried simply adding
static mesh components to the dungeon actor initially, but as I suspected, as soon as the maze was of any non-trivial
size the performance tanked, both in setting it up and when rendering it.

```cpp
UCLASS()
class DUNGEONCRAWLER_API ADungeon : public AActor
{
  GENERATED_BODY()
  
public:  
  ADungeon();

  void AddMesh(UInstancedStaticMeshComponent* Mesh, int X, int Y, float Rotation);
  void AddCellLabel(int X, int Y);

  UFUNCTION(BlueprintCallable)
  void GenerateMaze(int Width, int Height);

  UPROPERTY(EditAnywhere, BlueprintReadWrite)
  UInstancedStaticMeshComponent* FloorMesh;

  UPROPERTY(EditAnywhere, BlueprintReadWrite)
  UInstancedStaticMeshComponent* CeilingMesh;

  UPROPERTY(EditAnywhere, BlueprintReadWrite)
  UInstancedStaticMeshComponent* WallMesh;

  UPROPERTY(EditAnywhere, BlueprintReadWrite)
  UInstancedStaticMeshComponent* CornerConvexMesh;

  UPROPERTY(EditAnywhere, BlueprintReadWrite)
  UInstancedStaticMeshComponent* CornerConcaveMesh;

  UPROPERTY(EditAnywhere, BlueprintReadWrite)
  float TileSize;
};
```
The constructor creates the components, but setting the mesh for each *UInstancedStaticMeshComponent* is done
in a Blueprint class, *BP_Dungeon* that inherits from *ADungeon*.
```cpp
ADungeon::ADungeon()
{
  TileSize = 200.0f;
  RootComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Root"));
  FloorMesh = CreateDefaultSubobject<UInstancedStaticMeshComponent>(TEXT("Floor"));
  CeilingMesh = CreateDefaultSubobject<UInstancedStaticMeshComponent>(TEXT("Ceiling"));
  WallMesh = CreateDefaultSubobject<UInstancedStaticMeshComponent>(TEXT("Wall"));
  CornerConvexMesh = CreateDefaultSubobject<UInstancedStaticMeshComponent>(TEXT("CornerConvex"));
  CornerConcaveMesh = CreateDefaultSubobject<UInstancedStaticMeshComponent>(TEXT("CornerConcave"));
}
```
The *AddMesh* helper function adds an instance to the appropriate ISM, with the given rotation.
```cpp
void ADungeon::AddMesh(UInstancedStaticMeshComponent* Mesh, int X, int Y, float Rotation)
{
  FTransform Transform(FRotator(0.0f, Rotation, 0.0f), FVector(X, Y, 0.0f));
  Mesh->AddInstance(Transform);
}
```
The *GenerateMaze* function is embarrasingly long - I really should look into a different approach for
finding the appropriate mesh and rotation. I'm weighing that against the fact that I started this project
for getting a test environment for path finding and I really should get back to that :)
```cpp
void ADungeon::GenerateMaze(int Width, int Height)
{
  FloorMesh->ClearInstances();
  CeilingMesh->ClearInstances();
  WallMesh->ClearInstances();
  CornerConvexMesh->ClearInstances();
  CornerConcaveMesh->ClearInstances();
  
  FMazeGenerator M(Width, Height, 123);
  M.GenerateRecursive(1, 1);
  
  for (int Y = 0; Y < Height; Y++)
  {
    const float YPos = Y*3*TileSize;
    for (int X = 0; X < Width; X++)
    {
      FCell* Cell = M.GetCell(X, Y);
      FCell* NorthNeighbor = M.GetNeighbor(X, Y, North);
      FCell* EastNeighbor = M.GetNeighbor(X, Y, East);
      FCell* SouthNeighbor = M.GetNeighbor(X, Y, South);
      FCell* WestNeighbor = M.GetNeighbor(X, Y, West);

      const float XPos = X*3*TileSize;

      AddMesh(FloorMesh, XPos, YPos, 0.0f);
      if (Cell->Walls[North])
      {
        AddMesh(WallMesh, XPos, YPos - TileSize, 180.0f);
        if (Cell->Walls[West])
        {
          AddMesh(CornerConcaveMesh, XPos - TileSize, YPos - TileSize, 180.0f);
        } else
        {
          AddMesh(WallMesh, XPos - TileSize, YPos - TileSize, 180.0f);
        }
        if (Cell->Walls[East])
        {
          AddMesh(CornerConcaveMesh, XPos + TileSize, YPos - TileSize, 270.0f);
        } else
        {
          AddMesh(WallMesh, XPos + TileSize, YPos - TileSize, 180.0f);
        }
      } else
      {
        AddMesh(FloorMesh, XPos, YPos - TileSize, 0.0f);
        if (Cell->Walls[East])
        {
          if (EastNeighbor && EastNeighbor->Walls[North] || NorthNeighbor && NorthNeighbor->Walls[East])
          {
            AddMesh(WallMesh, XPos + TileSize, YPos - TileSize, 270.0f);
          } else
          {
            AddMesh(CornerConvexMesh, XPos + TileSize, YPos - TileSize, 270.0f);
          }
        } else
        {
          if (EastNeighbor && EastNeighbor->Walls[North])
          {
            AddMesh(CornerConvexMesh, XPos + TileSize, YPos - TileSize, 180.0f);
          } else
          {
            AddMesh(FloorMesh, XPos + TileSize, YPos - TileSize, 0.0f);
          }
        }
        if (Cell->Walls[West])
        {
          if (NorthNeighbor && NorthNeighbor->Walls[West] || WestNeighbor && WestNeighbor->Walls[North])
          {
            AddMesh(WallMesh, XPos - TileSize, YPos - TileSize, 90.0f);
          } else
          {
            AddMesh(CornerConvexMesh, XPos - TileSize, YPos - TileSize, 0.0f);
          }
        } else
        {
          if (WestNeighbor && WestNeighbor->Walls[North])
          {
            AddMesh(CornerConvexMesh, XPos - TileSize, YPos - TileSize, 90.0f);
          } else
          {
            AddMesh(FloorMesh, XPos - TileSize, YPos - TileSize, 0.0f);
          }
        }
      }

      if (Cell->Walls[South])
      {
        AddMesh(WallMesh, XPos, YPos + TileSize, 0.0f);
        if (Cell->Walls[West])
        {
          AddMesh(CornerConcaveMesh, XPos - TileSize, YPos + TileSize, 90.0f);
        } else
        {
          AddMesh(WallMesh, XPos - TileSize, YPos + TileSize, 0.0f);
        }
        if (Cell->Walls[East])
        {
          AddMesh(CornerConcaveMesh, XPos + TileSize, YPos + TileSize, 0.0f);
        } else
        {
          AddMesh(WallMesh, XPos + TileSize, YPos + TileSize, 0.0f);
        }
      } else
      {
        AddMesh(FloorMesh, XPos, YPos + TileSize, 0.0f);
        if (Cell->Walls[East])
        {
          if (SouthNeighbor && SouthNeighbor->Walls[East] || EastNeighbor && EastNeighbor->Walls[South])
          {
            AddMesh(WallMesh, XPos + TileSize, YPos + TileSize, 270.0f);
          } else
          {
            AddMesh(CornerConvexMesh, XPos + TileSize, YPos + TileSize, 180.0f);
          }
        }
        else
        {
          if (EastNeighbor && EastNeighbor->Walls[South])
          {
            AddMesh(CornerConvexMesh, XPos + TileSize, YPos + TileSize, 270.0f);
          } else
          {
            AddMesh(FloorMesh, XPos + TileSize, YPos + TileSize, 0.0f);
          }
        }
        if (Cell->Walls[West])
        {
          if (SouthNeighbor && SouthNeighbor->Walls[West] || WestNeighbor && WestNeighbor->Walls[South])
          {
            AddMesh(WallMesh, XPos - TileSize, YPos + TileSize, 90.0f);
          } else
          {
            AddMesh(CornerConvexMesh, XPos - TileSize, YPos + TileSize, 90.0f);
          }
        } else
        {
          if (WestNeighbor && WestNeighbor->Walls[South])
          {
            AddMesh(CornerConvexMesh, XPos - TileSize, YPos + TileSize, 0.0f);
          } else
          {
            AddMesh(FloorMesh, XPos - TileSize, YPos + TileSize, 0.0f);
          }
        }
      }

      if (Cell->Walls[West])
      {
        AddMesh(WallMesh, XPos - TileSize, YPos, 90.0f);
      } else
      {
        AddMesh(FloorMesh, XPos - TileSize, YPos, 0.0f);
      }
      if (Cell->Walls[East])
      {
        AddMesh(WallMesh, XPos + TileSize, YPos, 270.0f);
      } else
      {
        AddMesh(FloorMesh, XPos + TileSize, YPos, 0.0f);
      }
    }
  }
}
```
## What's next?
I'd like to try generating more interesting dungeons, rather than only generating perfect mazes. Bob Nystrom has an
[excellent blog](http://journal.stuffwithstuff.com/2014/12/21/rooms-and-mazes/) describing a procedural dungeon
generator, and [Wikipedia](https://en.wikipedia.org/wiki/Maze_generation_algorithm) describes other algorithms
for generating mazes. This is fun stuff, but I will probably have to focus more on the path finding aspect.