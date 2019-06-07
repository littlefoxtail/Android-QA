
# Chain of Responsibility Pattern(责任链模式)

避免请求发送者与接收者耦合在一起。让多个对象都有可能接收请求，将这些对象连接成一条链，并且沿着这条链传递请求，直到有对象处理它为止。
责任链模式结构的核心在于引入一个抽象处理者。

* Handler（抽象处理者）：它定义了一个处理请求的接口，一般设计为抽象类，由于不同的具体处理者处理请求的方式不同，因此在其中定义了抽象请求处理方法。因为每一个处理者的下家还是一个处理者，因此在抽象处理者中定义了一个抽象处理者类型的对象，作为其对下家的引用。通过该引用，处理者可以连成一条链。

    ```java
    public abstract class RequestHandler {
        private static final LOGGER = LoggerFactory.getLogger(RequestHandler.class);
        private RequestHandler next;

        public RequestHandler(RequestHandler next) {
            this.next = next;
        }
        public void handleRequest(Request req) {
            if (next != null) {
                next.handleRequest(req);
            }
        }

        protected void printHandling(Request req) {
            LOGGER.info("{} handling rquest\"{}\"", this, req);
        }

        @Override
        public abstract String toString();
    }

* ConcreteHandler（具体处理类）：它是抽象处理者的子类，可以处理用户请求，在具体处理者类中实现了抽象处理者中定义的抽象请求处理方法，在处理请求zhiIan需要进行判断，看是否有相应的处理权限，如果可以处理请求就处理它，否则将请求转发给后继者；在具体处理者中可以访问链中下一个对象，以便请求的转发。

    ```java
    public class OrcCommand extends RequestHandler {
        public OrcCommand(RequestHandler handler) {
            super(handler);
        }

        @Override
        public void handleRequest(Request req) {
            if (req.getRequestType().equals(RequestType.DEFEND_CASTLE)) {
                printHanling(req);
                req.markHandled();
            } else {
                super.handleRequest(req);
            }
        }

        @Override
        public String toString() {
            return "Orc commander";
        }
    }
    ```

    ```java
    public class OrcKing {
        RequestHandler chain;

        public OrcKing() {
            buildChain();
        }

        private void buildChain() {
            chain = new OrcCommander(new OrcOffice(new OrcSoldier(null)));
        }

        public void makeRequest(Request req) {
            chain.handleRequest(req);
        }
    }
    ```

    ```java
    public class Requst {
        private final RquestType requestType;
        private final String requestDescription;
        private boolean handled;

        public Request(final RquestType requestType, final String requestDecription) {
            this.requestType = Objects.requestNonNull(requestType);
            this.requestDescription = Object.requestNonNull(requestDescription);
        }

        public String getRequestDescription() { return requestDescription; }

        public RequestType getRequestType() { return requestType; }

        public void markHandled() { this.handled = true; }

        public boolean isHandled() { return this.handled; }

        @Override
        public String toString() { return getRequestDescription(); }
        }

        public enum RequestType {
            DEFEND_CASTLE, TORTURE_PRISONER, COLLECT_TAX
        }
    ```

在职责链模式里，很多对象由每一个对象对其下家的引用而连接起来形成一条链。请求在这个链上传递，直到链上的某一个对象决定处理此请求。
> 注意：职责链模式并不创建职责链，职责链的创建工作必须由系统其它部分来完成，一般是使用该职责链的客户端中创建职责链

## 职责链总结

通过建立一条链来组织请求的处理者，请求将沿着链进行传递，请求发送者无须知道请求在何时、何处以及如何被处理，实现了请求发送者与处理者的解耦

### 职责链优点

1. 使得一个对象无须知道是其他哪一个对象处理其请求，对象仅需知道该请求会被处理即可，接受者和发送者都没有对方的明确信息，且链中的对象不需要知道链的结构，由客户端负责链的创建，降低了系统的耦合度
2. 请求处理对象仅需维持一个指向其后继者的引用，而不需要维持它对所有的候选处理者的引用，可简化对象的相互连接。
3. 在给对象分派职责时，职责链可以给我们更多的灵活性，可以通过在运行时对该链进行动态的增加或修改来增加或改变处理一个请求的职责。
4. 在系统中增加一个新的具体请求处理者时无需修改原有系统的代码，只需要在客户端重新建链即可，符合“开闭原则”。

### 职责链缺点

1. 由于一个请求没有明确的接收者，那么就不能保证它一定会被处理，该请求可能一直到链的末端都得不到处理；一个请求也可能因职责链没有正确配置而得不到处理
2. 对于较长的职责链，请求的处理可能涉及到多个处理对象，系统性能将受到一定的影响，而且在进行代码调试时不太方便。
3. 如果建链不当，可能造成循环调用

### 职责链使用场景

1. 有多个对象可以处理同一个请求，具体哪个对象处理该请求待运行时刻再确定，客户端只需将请求提交到链上，而无需关心请求的处理对象是谁以及它是如何处理。
2. 在不明确指定接收者的情况下，向多个对象中的一个提交一个请求。
3. 可动态指定一组对象处理请求，客户端可以动态创建职责链来处理请求，还可以改变链中处理者的先后次序。
