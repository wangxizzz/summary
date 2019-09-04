1.**hashcode与equals方法重写的例子：**
```java
// 摘录自TopicPartition
public final class TopicPartition implements Serializable {
    private static final long serialVersionUID = -613627415771699627L;
    // 缓存hash值
    private int hash = 0;
    private final int partition;
    private final String topic;

    public TopicPartition(String topic, int partition) {
        this.partition = partition;
        this.topic = topic;
    }

    public int partition() {
        return partition;
    }

    public String topic() {
        return topic;
    }

    @Override
    public int hashCode() {
        if (hash != 0)
            return hash;
        final int prime = 31;
        int result = 1;
        result = prime * result + partition;
        result = prime * result + Objects.hashCode(topic);
        this.hash = result;
        return result;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj)
            return true;
        if (obj == null)
            return false;
        if (getClass() != obj.getClass())
            return false;
        TopicPartition other = (TopicPartition) obj;
        return partition == other.partition && Objects.equals(topic, other.topic);
    }

    @Override
    public String toString() {
        return topic + "-" + partition;
    }
}

// 摘录自Node
public class Node {

    private static final Node NO_NODE = new Node(-1, "", -1);

    private final int id;
    private final String idString;
    private final String host;
    private final int port;
    private final String rack;

    // Cache hashCode as it is called in performance sensitive parts of the code (e.g. RecordAccumulator.ready)
    private Integer hash;

    @Override
    public int hashCode() {
        Integer h = this.hash;
        if (h == null) {
            int result = 31 + ((host == null) ? 0 : host.hashCode());
            result = 31 * result + id;
            result = 31 * result + port;
            result = 31 * result + ((rack == null) ? 0 : rack.hashCode());
            this.hash = result;
            return result;
        } else {
            return h;
        }
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj)
            return true;
        if (obj == null || getClass() != obj.getClass())
            return false;
        Node other = (Node) obj;
        return (host == null ? other.host == null : host.equals(other.host)) &&
            id == other.id &&
            port == other.port &&
            (rack == null ? other.rack == null : rack.equals(other.rack));
    }

    @Override
    public String toString() {
        return host + ":" + port + " (id: " + idString + " rack: " + rack + ")";
    }


```

2.