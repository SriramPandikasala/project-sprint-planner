# Integrating DHTMLx Gantt with Server-Sent Events: A Real-Time Project Management Solution

## Introduction
In a world where project management relies solely on static spreadsheets and manual updates, overseeing multiple projects and their corresponding sprints in real-time would be a monumental challenge. This is where the versatility of Gantt charts shines. Gantt charts offer dynamic visual representations of project timelines, providing project managers with an effective means to track progress and make informed decisions.

Our challenge is to empower Group Project Managers to seamlessly monitor a multitude of projects and their associated sprints through an interactive Gantt chart view. This Gantt chart not only displays the project hierarchy but also updates in real-time whenever a Project Manager initiates changes within their purview. Gantt charts provide an intuitive way to visualize project timelines, dependencies, and progress, offering a comprehensive view of the entire project portfolio. However, achieving real-time synchronization in a web application, where multiple users interact with project data, presents complexities. Traditional approaches like manual refreshes or constant polling fall short of delivering the immediacy required in today's fast-paced project management landscape.

This is where Server-Sent Events (SSE) become our solution. SSE establishes a direct line of communication between the server and the client, enabling the server to push updates to the client as they occur. SSE eliminates the need for continuous polling, significantly reducing server load and latency. In the upcoming sections, we will delve deeper into how we've harnessed the capabilities of DHTMLx Gantt and leveraged Server-Sent Events (SSE) in our Angular-based project management solution. Together, they create a dynamic, real-time environment where project managers can effortlessly oversee projects and sprints, enabling them to make data-driven decisions with ease.

## DHTMLx Gantt and Event Source in This Context
Our technology stack consists of Angular for the user interface and Express.js in the backend. We use DHTMLx Gantt, a powerful JavaScript library for creating interactive Gantt charts. To establish real-time communication between the server and the client, we leverage SSE through Express.js.

 
## Code Structure
### Angular:
#### Services:
  * **GanttAPIService**:

    This service is responsible for interacting with the server to retrieve data. It calls the event source from Express and formats the received data into a specific format that can be easily consumed by the application.
  * **GanttDataService**:

    GanttDataService serves as a centralized data store for the application. Its primary responsibility is to store project and sprint data. It employs a BehaviorSubject to expose this data as an observable to other parts of the application.
  * **GanttCommunicatorService**:
    - The `GanttCommunicatorService` acts as a bridge between components and other services, namely `GanttAPIService` and `GanttDataService`. It injects these services and provides methods for coordinating data flow.
    - `fireForGanttData` method is responsible for initiating communication with the server through Server-Sent Events (SSE) and processing the data received from the server.
    -	`listenToGanttEvent` method ensures that the data updated by GanttDataService flows to the components at regular intervals, enabling real-time updates in the Gantt chart view.

#### Components:

* **Gantt-wrapper component**:
  - This component is responsible for rendering the Gantt chart view. It uses the DHTMLx Gantt library to display project and sprint data.
  - The Gantt-wrapper component relies on the GanttCommunicatorService to fetch and update data. It subscribes to the BehaviorSubject provided by GanttDataService to stay informed about data changes and updates the Gantt chart accordingly.
* **App component**:
  - The App component is the top-level component responsible for rendering the Gantt-wrapper component. It serves as the entry point for the Gantt chart view in the application.
 
### Express.js:
There is no user interface for project managers to add new projects. Instead, we simulate project additions using a dummy data generator.

The Express API endpoint is configured to initiate an EventSource. This EventSource pushes projects, one by one, at intervals of 3 seconds. Once all projects have been pushed, the event stream is closed.

This approach allows us to simulate real-time updates for the Gantt chart in the absence of a user interface for project creation.

## Implementation
Upon the initialization of the Gantt component, our application follows a specific set of steps to seamlessly integrate DHTMLx Gantt with Server-Sent Events (SSE). Here's a detailed walkthrough of how this is achieved:

```ts
  ngOnInit() {
    this.setConfigs();

    this.fetchProjects();
    this.listenToProjectUpdates()
  }

```




### Gantt Initialization: 
We start by initializing the DHTMLx Gantt chart within our Angular application. During this initialization phase, we define the columns and configure the chart to display project and sprint data.
```ts
  setConfigs() {
    gantt.config.date_format = '%Y-%m-%d %H:%i';
    gantt.init(this.ganttContainer.nativeElement);
    gantt.config.columns = this.createColumns();

    gantt.templates.task_class = function(start, end, task){
      return isChildTask(task) ? 'child-task': 'parent-task'
    };
  }
```


#### Invoking GanttCommunicatorService: 
After the Gantt chart is set up, we invoke the fireForGanttData method from the GanttCommunicatorService. This service plays a critical role in facilitating communication between our Angular components and the server.

```ts
  fetchProjects() {
    this.ganttCommunicatorService.fireForGanttData();
  }
 
  fireForGanttData() {
    if (this.sseSubs$) {
      this.sseSubs$.unsubscribe();
    }

   this.ganttApiService.handleSSE().subscribe({
      next: (data) => {
        this.ganttDataService.updateData(data)
      }
    });
  }

```



#### Listening to SSE: 
The heart of real-time updates lies in Server-Sent Events. When we invoke `fireForGanttData`, the service establishes a connection to the SSE stream sent from our Express.js backend. This connection acts as a continuous channel through which the server can send updates to the client.

```ts
    handleSSE(): Observable<any> {
        const unsubscribeFromSSE = ({
            sse$,
            subscriptionList,
        }: {
            sse$: EventSource;
            subscriptionList: Subscription;
        }) => {
            subscriptionList && subscriptionList.unsubscribe();
            sse$ && sse$.close();
        };
        const obs = new Observable((subscriber) => {
            const dummySSE = new EventSource(
                'http://localhost:8084/api/fetch-sse',
                {}
            );

            const subscriptionList = new Subscription();

            subscriptionList.add(
                fromEvent(dummySSE, 'project-detail-fetch')
                    .pipe(
                        map((resp: any) => {
                            console.log("Next Project Received");
                            return JSON.parse(resp.data)
                        }),
                        map((projectData) => {

                            return projectData.reduce(
                                (
                                    acc: {
                                        tasks: any[];
                                        links: any[];
                                    },
                                    ganttData
                                ) => {
                                    acc.links = [...acc.links, ...ganttData.links];
                                    acc.tasks.push(createProjectTaskObj(ganttData));

                                    ganttData.sprints.forEach((each) => {
                                        acc.tasks.push(createSprintTaskObj(each, ganttData.id));
                                    });

                                    return acc;
                                },
                                {
                                    tasks: [],
                                    links: [],
                                }
                            );
                        })
                    )
                    .subscribe({
                        next: (data) => {
                            subscriber.next(data);
                        },
                    })
            );

            subscriptionList.add(
                fromEvent(dummySSE, 'closestream-fetch')
                    .pipe(map((resp: any) => JSON.parse(resp.data)))
                    .subscribe({
                        next: (data) => {
                            console.log('close');

                            unsubscribeFromSSE({
                                sse$: dummySSE,
                                subscriptionList: subscriptionList,
                            });
                        },
                    })
            );

            return () => {
                console.log('SSE, closed');

                unsubscribeFromSSE({
                    sse$: dummySSE,
                    subscriptionList: subscriptionList,
                });
            };
        });

        return obs;
    }

```


#### Updating the BehaviorSubject: 

As the SSE stream receives updates from the server, the GanttAPIService processes this data and ensures it is in a suitable format for our application. This processed data is then used to update a BehaviorSubject located within the GanttDataService. A BehaviorSubject is a special type of Observable that allows components to receive real-time data updates.

```ts
export class GanttDataService {
    private data$: Subject<any> = new Subject();
    dataObs$ = this.data$.asObservable();

    constructor() {}

    updateData(data: {
        tasks: any,
        links: any
    }) {
        this.data$.next(data);
    }
}
```



#### Propagation of Data: 

The BehaviorSubject, which contains the processed data, emits an event whenever new data arrives. This event is listened to by the components subscribed to the BehaviorSubject, including our Gantt component. Consequently, whenever the BehaviorSubject is updated, the Gantt chart view is also updated accordingly.

#### Continual Updates: 

in the ganttcomponent, we initiate the listenToGanttEvent method from the GanttCommunicatorService. This method ensures that the data emitted by the BehaviorSubject, and thus, any updates from SSE, flows to the Gantt component at regular intervals. In our specific case, this update occurs every 3 seconds.

#### Updating the Gantt View: 
The Gantt component, upon receiving new data from the BehaviorSubject, uses the gantt.parse() method provided by DHTMLx Gantt to update the chart view. This allows project managers to visualize changes to projects and sprints in real-time, providing them with immediate insights and the ability to make prompt decisions.

```ts
  updateGanttView(projectData) {
    gantt.parse({ data: projectData.tasks, links: projectData.links });
  }

  listenToProjectUpdates() {
    this.ganttCommunicatorService.listenToGanttData().subscribe({
      next: data => {
        this.updateGanttView(data);
      }
    })
  }
```

## Conclusion

Our implementation of integrating DHTMLx Gantt with Server-Sent Events offers several benefits for real-time project management:

- Instant Updates: SSE ensures that any changes made by project managers are instantly reflected in the Gantt chart, providing real-time visibility into project progress.
- Reduced Server Load: SSE eliminates the need for constant polling, reducing server load and improving overall application performance.
- Efficient Communication: The use of SSE streamlines communication between the server and the client, resulting in a smoother user experience.
