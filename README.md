# Swift Sharing Exploration

This repository is a workspace for exploring Swift Sharing, SQLite, and Firestore integrations, including App Group extensions.

## Architecture

This project is organized as a root container with Git Submodules for the various components and dependencies.

### Modules

- **[swift-sharing-extension-exploration](https://github.com/technoplato/swift-sharing-extension-exploration)**: Core exploration code, including the App Group extension and transcription experiments.
- **[sharing-firestore](https://github.com/bitkey-oss/sharing-firestore)**: Dependency for Firestore integration.
- **[sqlite-data](https://github.com/pointfreeco/sqlite-data)**: Dependency for SQLite data handling.
- **[swift-sharing](https://github.com/pointfreeco/swift-sharing)**: Core Swift Sharing library.

## Setup Instructions

### 1. Cloning the Repository

To set this up on a new machine, you must clone recursively to fetch all submodules:

```bash
git clone --recursive https://github.com/technoplato/swift-sharing-exploration-root.git
```

If you have already cloned it without `--recursive`, you can initialize the submodules manually:

```bash
git submodule update --init --recursive
```

### 2. Building

1. Open `swift-sharing-extension-exploration/swift-sharing-and-sqlite-data-exploration-with-extensions.xcodeproj` in Xcode.
2. Ensure you have a valid Development Team selected for signing in the project settings.
3. Build and Run the scheme `swift-sharing-and-sqlite-data-exploration-with-extensions`.

## Notes

- This project uses **App Groups**. Ensure your provisioning profiles support the configured App Group identifiers if you are deploying to a device.
