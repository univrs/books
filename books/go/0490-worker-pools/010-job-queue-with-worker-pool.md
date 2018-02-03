Title: Job Queue with Worker Pool
Id: 18325
Score: 1
Body:
A job queue that maintains a worker pool, useful for doing things like background processing in web servers:

    package main

    import (
      "fmt"
      "runtime"
      "strconv"
      "sync"
      "time"
    )

    // Job - interface for job processing
    type Job interface {
      Process()
    }

    // Worker - the worker threads that actually process the jobs
    type Worker struct {
      done             sync.WaitGroup
      readyPool        chan chan Job
      assignedJobQueue chan Job

      quit chan bool
    }

    // JobQueue - a queue for enqueueing jobs to be processed
    type JobQueue struct {
      internalQueue     chan Job
      readyPool         chan chan Job
      workers           []*Worker
      dispatcherStopped sync.WaitGroup
      workersStopped    sync.WaitGroup
      quit              chan bool
    }

    // NewJobQueue - creates a new job queue
    func NewJobQueue(maxWorkers int) *JobQueue {
      workersStopped := sync.WaitGroup{}
      readyPool := make(chan chan Job, maxWorkers)
      workers := make([]*Worker, maxWorkers, maxWorkers)
      for i := 0; i < maxWorkers; i++ {
        workers[i] = NewWorker(readyPool, workersStopped)
      }
      return &JobQueue{
        internalQueue:     make(chan Job),
        readyPool:         readyPool,
        workers:           workers,
        dispatcherStopped: sync.WaitGroup{},
        workersStopped:    workersStopped,
        quit:              make(chan bool),
      }
    }

    // Start - starts the worker routines and dispatcher routine
    func (q *JobQueue) Start() {
      for i := 0; i < len(q.workers); i++ {
        q.workers[i].Start()
      }
      go q.dispatch()
    }

    // Stop - stops the workers and sispatcher routine
    func (q *JobQueue) Stop() {
      q.quit <- true
      q.dispatcherStopped.Wait()
    }

    func (q *JobQueue) dispatch() {
      q.dispatcherStopped.Add(1)
      for {
        select {
        case job := <-q.internalQueue: // We got something in on our queue
          workerChannel := <-q.readyPool // Check out an available worker
          workerChannel <- job           // Send the request to the channel
        case <-q.quit:
          for i := 0; i < len(q.workers); i++ {
            q.workers[i].Stop()
          }
          q.workersStopped.Wait()
          q.dispatcherStopped.Done()
          return
        }
      }
    }

    // Submit - adds a new job to be processed
    func (q *JobQueue) Submit(job Job) {
      q.internalQueue <- job
    }

    // NewWorker - creates a new worker
    func NewWorker(readyPool chan chan Job, done sync.WaitGroup) *Worker {
      return &Worker{
        done:             done,
        readyPool:        readyPool,
        assignedJobQueue: make(chan Job),
        quit:             make(chan bool),
      }
    }

    // Start - begins the job processing loop for the worker
    func (w *Worker) Start() {
      go func() {
        w.done.Add(1)
        for {
          w.readyPool <- w.assignedJobQueue // check the job queue in
          select {
          case job := <-w.assignedJobQueue: // see if anything has been assigned to the queue
            job.Process()
          case <-w.quit:
            w.done.Done()
            return
          }
        }
      }()
    }

    // Stop - stops the worker
    func (w *Worker) Stop() {
      w.quit <- true
    }

    //////////////// Example //////////////////

    // TestJob - holds only an ID to show state
    type TestJob struct {
      ID string
    }

    // Process - test process function
    func (t *TestJob) Process() {
      fmt.Printf("Processing job '%s'\n", t.ID)
      time.Sleep(1 * time.Second)
    }

    func main() {
      queue := NewJobQueue(runtime.NumCPU())
      queue.Start()
      defer queue.Stop()

      for i := 0; i < 4*runtime.NumCPU(); i++ {
        queue.Submit(&TestJob{strconv.Itoa(i)})
      }
    }

|======|