# Building-Ground Relationship (BGR) Graph Classification

## Overview

The goal is to represent a building as a **graph** — where each spatial cell (ground slab, columns, offices, core) becomes a **node**, and shared faces between cells become **edges** — and then classify the overall building-ground relationship into one of five categories:

| Value | Label |
|-------|-------|
| 0 | Separation |
| 1 | Separation with Plinth |
| 2 | Adherence |
| 3 | Adherence with Plinth |
| 4 | Interlock |

The pipeline spans two notebooks:

- **Part 1 — S06-13A:** Building the graph from Rhino geometry
- **Part 2 — S06-13B:** Predicting the graph label using a pre-trained model

---

## Part 1 — Building the Graph

### Rhino Model

The building is modelled in Rhino. The building has a rectangular shaper (20x70 m) with 9 floors. The first floors is elevated by columns. The model has the following features:
- **54 ground polysurfaces
- **20 columns polysurfaces
- **9 cores polysurfaces
- **104 offices polysurfaces

Before running the notebook, the building must be modelled in **Rhino** and exported as **four separate OBJ files**, each corresponding to a spatial layer:

| File | Spatial Element |
|------|----------------|
| `ground.obj` | Ground slab / podium |
| `columns.obj` | Structural columns |
| `offices.obj` | Office volumes |
| `core.obj` | Core and corridors |

![Imported building geometry from Rhino OBJ files](building.png)

![Color-coded cells after category assignment — green: ground, yellow: offices, blue: columns, red: core](color-coded.png)

---

### Step 1 — Import & Tag

The four OBJ files are loaded as topological objects using `Topology.ByOBJPath`. Each file becomes a list of geometry objects that will later be tagged and merged.

---

### Step 2 — Assign Categories (Selectors)

Each spatial element is assigned a **semantic tag** via a dictionary attached to an internal vertex (a *selector*). The selector acts as a pointer that carries the cell's identity metadata:

- `cell_type` — integer category (0–4)
- `cell_name` — human-readable string (`"ground"`, `"office"`, etc.)
- `cell_color` — display color for visualization

The process for each element type:
1. Extract faces from the OBJ objects
2. Flatten the face lists and merge them into a single topology (`SelfMerge`)
3. Extract closed volumetric cells from the merged topology
4. For each cell, create an internal vertex and attach a dictionary to it

---

### Step 3 — Build the CellComplex

All individual cells from every layer are merged into a single unified **CellComplex** — a topological structure that knows which cells share faces. This is the spatial model of the building.

---

### Step 4 — Transfer Dictionaries

The semantic tags created in Step 2 are transferred from the selectors onto the actual cells of the CellComplex. `TransferDictionariesBySelectors` matches each selector's internal vertex to the cell it sits inside, copying the dictionary onto that cell.

After this step, every cell in `model` carries its `cell_type`, `cell_name`, and `cell_color`. The result can be verified visually with `Topology.Show(..., faceColorKey="cell_color")`.

---

### Step 5 — Build the Adjacency Graph + One-Hot Features

**What happens:** The CellComplex is converted into a **graph** where:
- Each **node** = one spatial cell
- Each **edge** = a shared face between two adjacent cells

Each node then receives **one-hot encoded feature vectors** derived from its `cell_type`. One-hot encoding converts the integer category into a binary vector of length 5, with a `1` in the position corresponding to the category and `0` everywhere else:

```
cell_type = 3 (office)  →  [0, 0, 0, 1, 0]
                              ↑  ↑  ↑  ↑  ↑
                             f0 f1 f2 f3 f4
```

These five values are stored as `feature_00` through `feature_04` on each vertex's dictionary. The model uses these features as node-level input signals during classification.

---

### Step 6 — Export to CSV

The annotated graph is exported to three CSV files that serve as input for the prediction notebook.

| File | Contents |
|------|----------|
| `graphs.csv` | One row per graph — graph ID and manually assigned label |
| `nodes.csv` | One row per node — node ID, graph ID, label, and features |
| `edges.csv` | One row per edge — source and destination node IDs |

---

## Part 2 — Predicting with the Pre-Trained Model

### Load the Pre-Trained Model


The model was trained on the full Building-Ground Relationship dataset and has already learned to associate graph topology and node feature distributions with the five relationship categories.

---

### Predict and Inspect Results

The results are organised into a DataFrame with five columns:

| Column | Description |
|--------|-------------|
| `Actual Value` | The label you manually assigned (0–4) |
| `Predicted Value` | The label the model assigned (0–4) |
| `Actual Label` | Your label as a human-readable string |
| `Predicted Label` | The model's label as a human-readable string |
| `Confidence` | The model's certainty for its prediction (0.0–1.0) |

**Confidence** is derived from the maximum value in the model's output probability distribution across all five classes. A confidence of `1.0` means the model assigned essentially all probability mass to a single class — it was maximally certain. A confidence of `0.5` or lower suggests the model was uncertain between multiple classes.

> **Note:** High confidence does not guarantee a correct prediction. A confident wrong prediction (e.g. confidence `1.0` but `Actual ≠ Predicted`) often indicates that the  building shares strong structural features with a different category, or that the two categories are architecturally similar in ways the model has learned to conflate.

```