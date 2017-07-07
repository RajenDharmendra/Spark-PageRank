# Spark-PageRank

As an example of a more involved algorithm that can benefit from RDD partitioning, we consider PageRank. The PageRank algorithm, named after Google’s Larry Page, aims to assign a measure of importance (a “rank”) to each document in a set based on how many documents have links to it. It can be used to rank web pages, of course, but also scientific articles, or influential users in a social network.

PageRank is an iterative algorithm that performs many joins, so it is a good use case for RDD partitioning. The algorithm maintains two datasets: one of (pageID, linkList) elements containing the list of neighbors of each page, and one of (pageID, rank) elements containing the current rank for each page. It proceeds as follows:

    Initialize each page’s rank to 1.0.
    
    

    On each iteration, have page p send a contribution of rank(p)/numNeighbors(p) to its neighbors (the pages it has links to).
    
    

    Set each page’s rank to 0.15 + 0.85 * contributionsReceived.

The last two steps repeat for several iterations, during which the algorithm will converge to the correct PageRank value for each page. In practice, it’s typical to run about 10 iterations.

        // Assume that our neighbor list was saved as a Spark objectFile
        val links = sc.objectFile[(String, Seq[String])]("links")
                      .partitionBy(new HashPartitioner(100))
                      .persist()

        // Initialize each page's rank to 1.0; since we use mapValues, the resulting RDD
        // will have the same partitioner as links
        var ranks = links.mapValues(v => 1.0)

        // Run 10 iterations of PageRank
        for (i <- 0 until 10) {
          val contributions = links.join(ranks).flatMap {
            case (pageId, (links, rank)) =>
              links.map(dest => (dest, rank / links.size))
          }
          ranks = contributions.reduceByKey((x, y) => x + y).mapValues(v => 0.15 + 0.85*v)
        }

        // Write out the final ranks
        ranks.saveAsTextFile("ranks")
        
That’s it! The algorithm starts with a ranks RDD initialized at 1.0 for each element, and keeps updating the ranks variable on each iteration. The body of PageRank is pretty simple to express in Spark: it first does a join() between the current ranks RDD and the static links one, in order to obtain the link list and rank for each page ID together, then uses this in a flatMap to create “contribution” values to send to each of the page’s neighbors. We then add up these values by page ID (i.e., by the page receiving the contribution) and set that page’s rank to 0.15 + 0.85 * contributionsReceived.
