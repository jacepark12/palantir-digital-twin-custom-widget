# Palantir Digital Twin Workshop Widget

## Core Features

- **3D Model Viewer**: 3D model visualization through Autodesk Forge Viewer
- **Extension System**: Extensible feature modules using OSDK
- **Workshop Integration**: Bidirectional data synchronization with Palantir Workshop

## Demo

### Load Inventor Model in Palantir Workshop
![Demo1](./images/demo1.PNG)

### Render Ontology Data inside the viewer
![Demo2](./images/demo1.PNG)

## Workshop Configuration

The widget's Workshop integration is configured through `src/config.ts`. This file defines the data fields that the widget can exchange with Palantir Workshop. When extending the widget's functionality, you'll need to modify this configuration file.

### Current Configuration Fields

The current implementation includes the following fields as examples:

- **`stringField`**: A string input/output field with default value "test"
- **`selectedPartIdField`**: A number field for storing the selected part ID when clicking on 3D model
- **`selectedXCordField`**: A number field for storing the X coordinate of the clicked position
- **`selectedYCordField`**: A number field for storing the Y coordinate of the clicked position
- **`selectedZCordField`**: A number field for storing the Z coordinate of the clicked position
- **`focusedPartIds`**: A number list field for managing selected part IDs (defaults to empty array)

### Extending Configuration

To add new functionality that integrates with Workshop, you need to:

1. **Add new field definitions** to the `CONFIG` array in `src/config.ts`
2. **Update the widget logic** to read from and write to these new fields
3. **Modify the Workshop context** in your React components to access the new fields

Example of adding a new field:

```typescript
{
  fieldId: "newFeatureField",
  field: {
    type: "single",
    fieldValue: {
      type: "inputOutput",
      variableType: {
        type: "string",
        defaultValue: "defaultValue",
      },
    },
    label: "New Feature Field",
  },
}
```

### Configuration Usage

The current fields are used for:
- **Bidirectional Data Sync**: Selected parts in the 3D viewer are synchronized with the `focusedPartIds` field
- **Issue Tracking**: When users click on the 3D model, the part ID and coordinates are automatically stored in the respective fields
- **Workshop Integration**: The widget can read and write data to Workshop variables, enabling integration with other Workshop components

## Extension Development Guide

This project is designed to allow Extension development using OSDK (Open Source Development Kit). Extensions are modular components that add new functionality to the 3D viewer.

### Extension Basic Structure

Extensions are implemented by extending `Autodesk.Viewing.Extension`:

```typescript
class MyExtension extends (window as any).Autodesk.Viewing.Extension {
  load() {
    // Extension initialization logic
    return true;
  }

  unload() {
    // Extension cleanup logic
    return true;
  }
}

// Extension registration
(window as any).Autodesk.Viewing.theExtensionManager.registerExtension('MyExtension', MyExtension);
```

### Heatmap Extension Example

Let's explore Extension development methods through the currently implemented Heatmap Extension:

#### 1. Extension Class Definition

```typescript
class HeatmapExtension extends (window as any).Autodesk.Viewing.Extension {
  private _enabled = false;
  private _intensity = 0.5;
  private _type: HeatmapType = HEATMAP_TYPES[0];
  private _button: any = null;
  private _toolbar: any = null;
  private _panel: HeatmapPanel | null = null;
  private _data: any[] = [];
  private _unsubscribe: (() => void) | null = null;
}
```

#### 2. Extension Load/Unload

```typescript
load() {
  const viewer = (this as any).viewer;

  if (viewer.toolbar) {
    this.createUI();
  } else {
    // Register event listener if toolbar hasn't been created yet
    const onToolbarCreated = () => {
      this.createUI();
      viewer.removeEventListener(
        (window as any).Autodesk.Viewing.TOOLBAR_CREATED_EVENT,
        onToolbarCreated
      );
    };
    viewer.addEventListener(
      (window as any).Autodesk.Viewing.TOOLBAR_CREATED_EVENT,
      onToolbarCreated
    );
  }
  return true;
}

unload() {
  const viewer = (this as any).viewer;
  if (this._toolbar) {
    viewer.toolbar.removeControl(this._toolbar);
    this._toolbar = null;
  }
  if (this._enabled) {
    this._removeColors();
  }
  if (this._unsubscribe) {
    this._unsubscribe();
    this._unsubscribe = null;
  }
  return true;
}
```

#### 3. UI Component Creation

```typescript
private createUI() {
  const viewer = (this as any).viewer;

  // Create panel
  this._panel = new HeatmapPanel(viewer, viewer.container, (change: any) => {
    if (change.intensity !== undefined) {
      this._intensity = change.intensity;
    }
    if (change.type !== undefined) {
      this._type = change.type;
    }
    if (this._enabled) {
      this._applyColors();
    }
  });

  // Create button
  this._button = new (window as any).Autodesk.Viewing.UI.Button('HeatmapButton');
  this._button.onClick = async () => {
    this._enabled = !this._enabled;
    if (this._panel) {
      this._panel.setVisible(this._enabled);
    }
    if (this._enabled) {
      await this._applyColors();
      this._button.setState(0);
    } else {
      this._removeColors();
      this._button.setState(1);
    }
    viewer.impl.invalidate(true, true, true);
  };

  // Add to toolbar
  this._toolbar = viewer.toolbar.getControl('CustomToolbar') ||
    new (window as any).Autodesk.Viewing.UI.ControlGroup('CustomToolbar');
  this._toolbar.addControl(this._button);
  viewer.toolbar.addControl(this._toolbar);
}
```

#### 4. Data Processing and Visualization

```typescript
private async _applyColors() {
  const viewer = (this as any).viewer;

  try {
    this._data = await loadHeatmapData(this._type);

    for (const item of this._data) {
      const color = new (window as any).THREE.Color();
      color.setHSL(item.heat * 0.33, 1.0, 0.5);
      viewer.setThemingColor(item.id, new (window as any).THREE.Vector4(color.r, color.g, color.b, this._intensity));
    }

    // Subscribe to real-time updates
    if (this._unsubscribe) {
      this._unsubscribe();
    }
    this._unsubscribe = subscribeHeatmapData(this._type, {
      onChange: (data) => {
        this._data = data;
        this._applyColors();
      },
      onError: () => {
        console.error('Error in heatmap subscription');
      }
    });
  } catch (error) {
    console.error('Error applying heatmap colors:', error);
  }
}
```

### Extension Development Steps

1. **Create Extension Class**: Define class extending `Autodesk.Viewing.Extension`
2. **Implement Required Methods**: Implement `load()` and `unload()` methods
3. **Develop UI Components**: Create necessary UI elements like buttons, panels, sliders
4. **Implement Core Logic**: Implement the core functionality of the Extension
5. **Handle Events**: Process viewer events and user interactions
6. **Register Extension**: Register Extension using `registerExtension`

### Extension Registration

To register an Extension, use the following code:

```typescript
(window as any).Autodesk.Viewing.theExtensionManager.registerExtension('ExtensionName', ExtensionClass);
```

### Important Notes

- Extensions must be registered after the viewer is fully loaded
- UI components must be added after the toolbar is created
- All listeners and resources must be cleaned up in the `unload()` method to prevent memory leaks
- Use subscribe/unsubscribe patterns for real-time data updates

## Development Environment Setup

### Prerequisites

- Node.js 18+
- pnpm (package manager)
- Palantir Foundry account and token

### Installation and Running

```bash
# Install dependencies
pnpm install

# Run development server
pnpm run dev

# Build for production
pnpm run build
```

### Environment Variables

- `FOUNDRY_TOKEN`: Palantir Foundry authentication token
- `VITE_FOUNDRY_REDIRECT_URL`: OAuth redirect URL

