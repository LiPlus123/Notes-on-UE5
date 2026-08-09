# 网络游戏与帧同步

## 网络复制

```mermaid
classDiagram
    class AActor {
        #uint8 bReplicates
        -uint8 bReplicateMovement
        #TArray~UActorComponent*~ ReplicatedComponents

        void RemoveReplicatedComponent(UActorComponent* Component)
    }

    UWorld *-- UNetDriver

    class UWorld {
        UNetDriver* NetDriver
    }

    class UNetDriver {

    }
```

### Iris 复制系统